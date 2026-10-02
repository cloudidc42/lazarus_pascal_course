# Part 43 - Multithreading ใน Lazarus/Pascal

## บทนำ

Multithreading ช่วยให้โปรแกรมทำงานหลายอย่างพร้อมกัน เหมาะสำหรับงานที่ต้องใช้เวลานาน เช่น การดาวน์โหลดไฟล์ การประมวลผลข้อมูลขนาดใหญ่ หรือการทำงานกับ I/O โดยไม่หยุดการทำงานของ UI

---

## 43.1 Thread Concepts

### ทำความเข้าใจ Thread

```
Main Process
│
├── Thread 1 (Main/UI Thread)
│   ├── วน event loop
│   └── ตอบสนอง user interaction
│
├── Thread 2 (Worker Thread)
│   ├── ดาวน์โหลดไฟล์
│   └── ส่งผลลัพธ์กลับ UI
│
└── Thread 3 (Background Thread)
    ├── บันทึก log
    └── ทำ periodic tasks
```

### ประเด็นสำคัญ
- **Thread Safety**: ข้อมูลที่ใช้ร่วมกันต้องได้รับการป้องกัน
- **Race Condition**: เกิดเมื่อ threads หลายตัวแก้ไขข้อมูลเดียวกัน
- **Deadlock**: เกิดเมื่อ threads รอกันเองเป็น cycle
- **Synchronization**: กลไกที่ใช้ควบคุมการเข้าถึงข้อมูล

---

## 43.2 TThread Class

### การสร้าง Thread พื้นฐาน

```pascal
program BasicThread;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils;

type
  TMyThread = class(TThread)
  private
    FMessage: string;
    FCount: integer;
  protected
    procedure Execute; override;
  public
    constructor Create(const Msg: string; Count: integer);
  end;

constructor TMyThread.Create(const Msg: string; Count: integer);
begin
  // FreeOnTerminate = True จะ free thread อัตโนมัติเมื่อ Execute จบ
  inherited Create(True); // True = create suspended
  FMessage := Msg;
  FCount := Count;
  FreeOnTerminate := True;
end;

procedure TMyThread.Execute;
var
  i: integer;
begin
  for i := 1 to FCount do
  begin
    if Terminated then Break; // ตรวจสอบว่าถูกสั่งหยุดหรือไม่
    
    // ทำงาน
    WriteLn(Format('[Thread %s] Step %d/%d', [FMessage, i, FCount]));
    Sleep(500); // simulate work
  end;
  
  WriteLn('[Thread ', FMessage, '] Done!');
end;

var
  Thread1, Thread2, Thread3: TMyThread;
begin
  WriteLn('Starting threads...');
  WriteLn('');
  
  // สร้าง threads
  Thread1 := TMyThread.Create('Alpha', 5);
  Thread2 := TMyThread.Create('Beta', 4);
  Thread3 := TMyThread.Create('Gamma', 6);
  
  // เริ่มทำงาน
  Thread1.Start;
  Thread2.Start;
  Thread3.Start;
  
  // รอจนทุก thread จบ
  Thread1.WaitFor;
  Thread2.WaitFor;
  Thread3.WaitFor;
  
  WriteLn('');
  WriteLn('All threads completed!');
end.
```

---

## 43.3 Execute Method

### Execute Method แบบต่างๆ

```pascal
program ExecuteMethod;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Math;

type
  // Thread คำนวณ prime numbers
  TPrimeThread = class(TThread)
  private
    FStart: int64;
    FEnd: int64;
    FPrimes: TList;
    FLock: TCriticalSection;
    
    function IsPrime(N: int64): boolean;
  protected
    procedure Execute; override;
  public
    constructor Create(StartNum, EndNum: int64; SharedList: TList; Lock: TCriticalSection);
    destructor Destroy; override;
    
    property Primes: TList read FPrimes;
  end;

constructor TPrimeThread.Create(StartNum, EndNum: int64; 
  SharedList: TList; Lock: TCriticalSection);
begin
  inherited Create(True);
  FStart := StartNum;
  FEnd := EndNum;
  FPrimes := SharedList;
  FLock := Lock;
  FreeOnTerminate := False;
end;

destructor TPrimeThread.Destroy;
begin
  inherited Destroy;
end;

function TPrimeThread.IsPrime(N: int64): boolean;
var
  i: int64;
begin
  if N < 2 then Exit(False);
  if N = 2 then Exit(True);
  if N mod 2 = 0 then Exit(False);
  
  i := 3;
  while i * i <= N do
  begin
    if N mod i = 0 then Exit(False);
    Inc(i, 2);
  end;
  
  Result := True;
end;

procedure TPrimeThread.Execute;
var
  n: int64;
  PrimePtr: PInt64;
begin
  for n := FStart to FEnd do
  begin
    if Terminated then Break;
    
    if IsPrime(n) then
    begin
      // เพิ่มเข้า shared list ด้วย critical section
      FLock.Acquire;
      try
        New(PrimePtr);
        PrimePtr^ := n;
        FPrimes.Add(PrimePtr);
      finally
        FLock.Release;
      end;
    end;
  end;
end;

var
  SharedPrimes: TList;
  Lock: TCriticalSection;
  Threads: array[0..3] of TPrimeThread;
  i, j: integer;
  Range: integer;
  PrimeCount: integer;
  StartTime, EndTime: TDateTime;
begin
  Range := 1000;
  SharedPrimes := TList.Create;
  Lock := TCriticalSection.Create;
  
  try
    WriteLn('Finding primes from 1 to ', Range * 4, ' using 4 threads...');
    StartTime := Now;
    
    // แบ่งงานให้ 4 threads
    for i := 0 to 3 do
    begin
      Threads[i] := TPrimeThread.Create(
        i * Range + 1,
        (i + 1) * Range,
        SharedPrimes,
        Lock
      );
      Threads[i].Start;
    end;
    
    // รอทุก thread จบ
    for i := 0 to 3 do
      Threads[i].WaitFor;
    
    EndTime := Now;
    
    // นับจำนวน primes ที่พบ
    PrimeCount := SharedPrimes.Count;
    WriteLn('Found ', PrimeCount, ' prime numbers');
    WriteLn('Time: ', FormatDateTime('s.zzz', EndTime - StartTime), ' seconds');
    
    // แสดง primes แรก 20 ตัว
    WriteLn('First 20 primes found (unsorted):');
    Write('  ');
    j := 0;
    for i := 0 to Min(19, PrimeCount - 1) do
    begin
      Write(PInt64(SharedPrimes[i])^, ' ');
      Inc(j);
      if j mod 10 = 0 then
      begin
        WriteLn;
        Write('  ');
      end;
    end;
    WriteLn;
    
  finally
    // Cleanup
    for i := 0 to 3 do
    begin
      Threads[i].Free;
    end;
    
    for i := 0 to SharedPrimes.Count - 1 do
      Dispose(PInt64(SharedPrimes[i]));
    
    SharedPrimes.Free;
    Lock.Free;
  end;
end.
```

---

## 43.4 Synchronize Method

### การใช้ Synchronize สำหรับ UI Updates

```pascal
program SynchronizeMethod;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils;

type
  // Thread ที่ต้อง update UI
  TProgressThread = class(TThread)
  private
    FProgress: integer;
    FStatusMsg: string;
    FTotalSteps: integer;
    
    // Method นี้จะ run บน main thread
    procedure UpdateProgress;
    procedure UpdateStatus;
    procedure TaskCompleted;
  protected
    procedure Execute; override;
  public
    constructor Create(Steps: integer);
  end;

constructor TProgressThread.Create(Steps: integer);
begin
  inherited Create(True);
  FTotalSteps := Steps;
  FProgress := 0;
  FreeOnTerminate := False;
end;

procedure TProgressThread.UpdateProgress;
begin
  // Method นี้ execute บน main thread ผ่าน Synchronize
  WriteLn(Format('[Main Thread] Progress: %d%%', [FProgress]));
end;

procedure TProgressThread.UpdateStatus;
begin
  WriteLn('[Main Thread] Status: ', FStatusMsg);
end;

procedure TProgressThread.TaskCompleted;
begin
  WriteLn('[Main Thread] Task completed!');
  WriteLn('[Main Thread] Updating UI...');
end;

procedure TProgressThread.Execute;
var
  i: integer;
begin
  FStatusMsg := 'Starting process...';
  Synchronize(@UpdateStatus);
  
  for i := 1 to FTotalSteps do
  begin
    if Terminated then Break;
    
    // ทำงานหนักบน background thread
    WriteLn(Format('[Background] Working on step %d/%d...', [i, FTotalSteps]));
    Sleep(200); // simulate work
    
    // คำนวณ progress
    FProgress := Round(i / FTotalSteps * 100);
    
    // Update UI ผ่าน Synchronize (main thread)
    Synchronize(@UpdateProgress);
    
    // Update status ทุก 25%
    if FProgress mod 25 = 0 then
    begin
      FStatusMsg := Format('Completed %d%%', [FProgress]);
      Synchronize(@UpdateStatus);
    end;
  end;
  
  // Task done
  Synchronize(@TaskCompleted);
end;

// การใช้งานแบบ Queue (เพิ่มใน Lazarus forms)
type
  TDataProcessThread = class(TThread)
  private
    FData: string;
    FResult: string;
    FOnResult: TNotifyEvent;
    
    procedure NotifyResult;
  protected
    procedure Execute; override;
  public
    constructor Create(const Data: string; OnResult: TNotifyEvent);
    property Result: string read FResult;
  end;

constructor TDataProcessThread.Create(const Data: string; OnResult: TNotifyEvent);
begin
  inherited Create(True);
  FData := Data;
  FOnResult := OnResult;
  FreeOnTerminate := True;
end;

procedure TDataProcessThread.NotifyResult;
begin
  if Assigned(FOnResult) then
    FOnResult(Self);
end;

procedure TDataProcessThread.Execute;
begin
  // Process data (background work)
  Sleep(1000); // simulate processing
  FResult := 'Processed: ' + UpperCase(FData);
  
  // Queue notification to main thread
  Queue(@NotifyResult); // Queue ต่างจาก Synchronize ตรงที่ไม่ block
end;

procedure OnDataReady(Sender: TObject);
begin
  WriteLn('[Callback on Main Thread] Result: ', 
    TDataProcessThread(Sender).Result);
end;

var
  ProgressThread: TProgressThread;
  DataThread: TDataProcessThread;
begin
  WriteLn('=== Synchronize Demo ===');
  WriteLn('');
  
  ProgressThread := TProgressThread.Create(8);
  try
    ProgressThread.Start;
    ProgressThread.WaitFor;
  finally
    ProgressThread.Free;
  end;
  
  WriteLn('');
  WriteLn('=== Queue Demo ===');
  
  // Queue demo - ส่งข้อมูลไปประมวลผล
  DataThread := TDataProcessThread.Create('hello world', @OnDataReady);
  DataThread.Start;
  
  // รอ thread จบ (FreeOnTerminate = True ดังนั้นไม่ควร Free เอง)
  Sleep(1500);
  
  WriteLn('Done!');
end.
```

---

## 43.5 Thread Safety

### การเขียน Thread-Safe Code

```pascal
program ThreadSafety;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils;

// ตัวอย่าง: Counter ที่ไม่ thread-safe
type
  TUnsafeCounter = class
  private
    FValue: integer;
  public
    procedure Increment;
    property Value: integer read FValue;
  end;

procedure TUnsafeCounter.Increment;
begin
  // Race condition! หลาย threads อ่าน/เขียน FValue พร้อมกัน
  FValue := FValue + 1;
end;

// ตัวอย่าง: Counter ที่ thread-safe
type
  TAtomicCounter = class
  private
    FValue: integer;
    FLock: TCriticalSection;
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Increment;
    procedure Decrement;
    function GetAndReset: integer;
    property Value: integer read FValue;
  end;

constructor TAtomicCounter.Create;
begin
  inherited Create;
  FLock := TCriticalSection.Create;
  FValue := 0;
end;

destructor TAtomicCounter.Destroy;
begin
  FLock.Free;
  inherited Destroy;
end;

procedure TAtomicCounter.Increment;
begin
  FLock.Acquire;
  try
    Inc(FValue);
  finally
    FLock.Release;
  end;
end;

procedure TAtomicCounter.Decrement;
begin
  FLock.Acquire;
  try
    Dec(FValue);
  finally
    FLock.Release;
  end;
end;

function TAtomicCounter.GetAndReset: integer;
begin
  FLock.Acquire;
  try
    Result := FValue;
    FValue := 0;
  finally
    FLock.Release;
  end;
end;

// Thread-safe Queue
type
  generic TThreadSafeQueue<T> = class
  private
    FItems: array of T;
    FHead, FTail, FCount: integer;
    FCapacity: integer;
    FLock: TCriticalSection;
  public
    constructor Create(Capacity: integer = 100);
    destructor Destroy; override;
    
    function Enqueue(const Item: T): boolean;
    function Dequeue(out Item: T): boolean;
    function TryPeek(out Item: T): boolean;
    
    property Count: integer read FCount;
    function IsEmpty: boolean;
    function IsFull: boolean;
  end;

constructor TThreadSafeQueue.Create(Capacity: integer);
begin
  inherited Create;
  FCapacity := Capacity;
  SetLength(FItems, Capacity);
  FHead := 0;
  FTail := 0;
  FCount := 0;
  FLock := TCriticalSection.Create;
end;

destructor TThreadSafeQueue.Destroy;
begin
  FLock.Free;
  inherited Destroy;
end;

function TThreadSafeQueue.IsEmpty: boolean;
begin
  Result := FCount = 0;
end;

function TThreadSafeQueue.IsFull: boolean;
begin
  Result := FCount = FCapacity;
end;

function TThreadSafeQueue.Enqueue(const Item: T): boolean;
begin
  FLock.Acquire;
  try
    if IsFull then
      Exit(False);
      
    FItems[FTail] := Item;
    FTail := (FTail + 1) mod FCapacity;
    Inc(FCount);
    Result := True;
  finally
    FLock.Release;
  end;
end;

function TThreadSafeQueue.Dequeue(out Item: T): boolean;
begin
  FLock.Acquire;
  try
    if IsEmpty then
      Exit(False);
      
    Item := FItems[FHead];
    FHead := (FHead + 1) mod FCapacity;
    Dec(FCount);
    Result := True;
  finally
    FLock.Release;
  end;
end;

function TThreadSafeQueue.TryPeek(out Item: T): boolean;
begin
  FLock.Acquire;
  try
    if IsEmpty then
      Exit(False);
    Item := FItems[FHead];
    Result := True;
  finally
    FLock.Release;
  end;
end;

// Test threads ที่ใช้ counter
type
  TCounterThread = class(TThread)
  private
    FCounter: TAtomicCounter;
    FName: string;
    FIncrements: integer;
  protected
    procedure Execute; override;
  public
    constructor Create(Counter: TAtomicCounter; const Name: string; Incs: integer);
  end;

constructor TCounterThread.Create(Counter: TAtomicCounter; const Name: string; Incs: integer);
begin
  inherited Create(True);
  FCounter := Counter;
  FName := Name;
  FIncrements := Incs;
  FreeOnTerminate := False;
end;

procedure TCounterThread.Execute;
var
  i: integer;
begin
  for i := 1 to FIncrements do
  begin
    FCounter.Increment;
    if i mod 100 = 0 then
      WriteLn('[', FName, '] Progress: ', i, '/', FIncrements);
  end;
end;

var
  Counter: TAtomicCounter;
  Threads: array[0..4] of TCounterThread;
  ExpectedTotal, ActualTotal: integer;
  i: integer;
begin
  Counter := TAtomicCounter.Create;
  ExpectedTotal := 0;
  
  try
    WriteLn('=== Thread Safety Demo ===');
    WriteLn('Starting 5 threads, each incrementing counter 200 times...');
    
    for i := 0 to 4 do
    begin
      Threads[i] := TCounterThread.Create(Counter, 'T' + IntToStr(i + 1), 200);
      Threads[i].Start;
      Inc(ExpectedTotal, 200);
    end;
    
    for i := 0 to 4 do
      Threads[i].WaitFor;
    
    ActualTotal := Counter.Value;
    
    WriteLn('');
    WriteLn('Expected total: ', ExpectedTotal);
    WriteLn('Actual total: ', ActualTotal);
    
    if ActualTotal = ExpectedTotal then
      WriteLn('Thread safety: PASSED!')
    else
      WriteLn('Thread safety: FAILED! (Race condition occurred)');
      
  finally
    for i := 0 to 4 do
      Threads[i].Free;
    Counter.Free;
  end;
end.
```

---

## 43.6 Critical Sections

### การใช้ Critical Section

```pascal
program CriticalSectionDemo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils;

type
  TSharedResource = class
  private
    FLock: TCriticalSection;
    FData: TStringList;
    FAccessCount: integer;
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure AddData(const Value: string);
    function ReadData(Index: integer): string;
    function GetCount: integer;
    procedure Display;
    
    property AccessCount: integer read FAccessCount;
  end;

constructor TSharedResource.Create;
begin
  inherited Create;
  FLock := TCriticalSection.Create;
  FData := TStringList.Create;
  FAccessCount := 0;
end;

destructor TSharedResource.Destroy;
begin
  FData.Free;
  FLock.Free;
  inherited Destroy;
end;

procedure TSharedResource.AddData(const Value: string);
begin
  FLock.Acquire; // เข้า critical section
  try
    FData.Add(Value);
    Inc(FAccessCount);
    Sleep(10); // simulate some work
  finally
    FLock.Release; // ออก critical section เสมอ
  end;
end;

function TSharedResource.ReadData(Index: integer): string;
begin
  FLock.Acquire;
  try
    Inc(FAccessCount);
    if (Index >= 0) and (Index < FData.Count) then
      Result := FData[Index]
    else
      Result := '';
  finally
    FLock.Release;
  end;
end;

function TSharedResource.GetCount: integer;
begin
  FLock.Acquire;
  try
    Result := FData.Count;
  finally
    FLock.Release;
  end;
end;

procedure TSharedResource.Display;
var
  i: integer;
begin
  FLock.Acquire;
  try
    WriteLn('Data count: ', FData.Count);
    for i := 0 to FData.Count - 1 do
      WriteLn('  [', i, ']: ', FData[i]);
  finally
    FLock.Release;
  end;
end;

// Writer thread
type
  TWriterThread = class(TThread)
  private
    FResource: TSharedResource;
    FID: integer;
    FCount: integer;
  protected
    procedure Execute; override;
  public
    constructor Create(Resource: TSharedResource; ID, Count: integer);
  end;

constructor TWriterThread.Create(Resource: TSharedResource; ID, Count: integer);
begin
  inherited Create(True);
  FResource := Resource;
  FID := ID;
  FCount := Count;
  FreeOnTerminate := False;
end;

procedure TWriterThread.Execute;
var
  i: integer;
begin
  for i := 1 to FCount do
  begin
    if Terminated then Break;
    FResource.AddData(Format('Writer%d-Item%d', [FID, i]));
  end;
  WriteLn('[Writer', FID, '] Done, added ', FCount, ' items');
end;

var
  Resource: TSharedResource;
  Writers: array[0..2] of TWriterThread;
  i: integer;
begin
  WriteLn('=== Critical Section Demo ===');
  
  Resource := TSharedResource.Create;
  try
    // สร้าง 3 writer threads
    for i := 0 to 2 do
    begin
      Writers[i] := TWriterThread.Create(Resource, i + 1, 5);
      Writers[i].Start;
    end;
    
    // รอ writers จบ
    for i := 0 to 2 do
      Writers[i].WaitFor;
    
    WriteLn('');
    WriteLn('Final state:');
    Resource.Display;
    WriteLn('Total access count: ', Resource.AccessCount);
    
  finally
    for i := 0 to 2 do
      Writers[i].Free;
    Resource.Free;
  end;
end.
```

---

## 43.7 Mutexes

### การใช้ Mutex

```pascal
program MutexDemo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils
  {$IFDEF UNIX}, BaseUnix, PThreads{$ENDIF}
  {$IFDEF WINDOWS}, Windows{$ENDIF};

type
  // Cross-platform Mutex wrapper
  TMutex = class
  private
    {$IFDEF WINDOWS}
    FMutex: HANDLE;
    {$ENDIF}
    {$IFDEF UNIX}
    FMutex: pthread_mutex_t;
    {$ENDIF}
    FName: string;
  public
    constructor Create(const Name: string = '');
    destructor Destroy; override;
    
    procedure Acquire;
    function TryAcquire(TimeoutMs: cardinal = 0): boolean;
    procedure Release;
    
    property Name: string read FName;
  end;

constructor TMutex.Create(const Name: string);
begin
  inherited Create;
  FName := Name;
  {$IFDEF WINDOWS}
  if Name <> '' then
    FMutex := CreateMutex(nil, False, PChar(Name))
  else
    FMutex := CreateMutex(nil, False, nil);
  {$ENDIF}
  {$IFDEF UNIX}
  pthread_mutex_init(@FMutex, nil);
  {$ENDIF}
end;

destructor TMutex.Destroy;
begin
  {$IFDEF WINDOWS}
  CloseHandle(FMutex);
  {$ENDIF}
  {$IFDEF UNIX}
  pthread_mutex_destroy(@FMutex);
  {$ENDIF}
  inherited Destroy;
end;

procedure TMutex.Acquire;
begin
  {$IFDEF WINDOWS}
  WaitForSingleObject(FMutex, INFINITE);
  {$ENDIF}
  {$IFDEF UNIX}
  pthread_mutex_lock(@FMutex);
  {$ENDIF}
end;

function TMutex.TryAcquire(TimeoutMs: cardinal): boolean;
begin
  {$IFDEF WINDOWS}
  Result := WaitForSingleObject(FMutex, TimeoutMs) = WAIT_OBJECT_0;
  {$ENDIF}
  {$IFDEF UNIX}
  Result := pthread_mutex_trylock(@FMutex) = 0;
  {$ENDIF}
end;

procedure TMutex.Release;
begin
  {$IFDEF WINDOWS}
  ReleaseMutex(FMutex);
  {$ENDIF}
  {$IFDEF UNIX}
  pthread_mutex_unlock(@FMutex);
  {$ENDIF}
end;

// แทนที่จะสร้าง cross-platform mutex เอง ใน Lazarus ให้ใช้ TCriticalSection
// ซึ่ง wrap ไว้ให้แล้ว นี่คือตัวอย่างการใช้งาน

type
  TBankAccount = class
  private
    FBalance: double;
    FLock: TCriticalSection;
    FAccountNumber: string;
    FOwner: string;
  public
    constructor Create(const AccNum, Owner: string; InitialBalance: double);
    destructor Destroy; override;
    
    function Deposit(Amount: double): boolean;
    function Withdraw(Amount: double): boolean;
    function Transfer(Target: TBankAccount; Amount: double): boolean;
    function GetBalance: double;
    
    property AccountNumber: string read FAccountNumber;
    property Owner: string read FOwner;
  end;

constructor TBankAccount.Create(const AccNum, Owner: string; InitialBalance: double);
begin
  inherited Create;
  FAccountNumber := AccNum;
  FOwner := Owner;
  FBalance := InitialBalance;
  FLock := TCriticalSection.Create;
end;

destructor TBankAccount.Destroy;
begin
  FLock.Free;
  inherited Destroy;
end;

function TBankAccount.Deposit(Amount: double): boolean;
begin
  Result := False;
  if Amount <= 0 then Exit;
  
  FLock.Acquire;
  try
    FBalance := FBalance + Amount;
    WriteLn(Format('[%s] Deposit: +%.2f = %.2f', [FAccountNumber, Amount, FBalance]));
    Result := True;
  finally
    FLock.Release;
  end;
end;

function TBankAccount.Withdraw(Amount: double): boolean;
begin
  Result := False;
  if Amount <= 0 then Exit;
  
  FLock.Acquire;
  try
    if FBalance < Amount then
    begin
      WriteLn(Format('[%s] Insufficient funds: %.2f < %.2f', 
        [FAccountNumber, FBalance, Amount]));
      Exit;
    end;
    FBalance := FBalance - Amount;
    WriteLn(Format('[%s] Withdraw: -%.2f = %.2f', [FAccountNumber, Amount, FBalance]));
    Result := True;
  finally
    FLock.Release;
  end;
end;

function TBankAccount.Transfer(Target: TBankAccount; Amount: double): boolean;
begin
  Result := False;
  
  // เพื่อป้องกัน deadlock ต้อง lock ตามลำดับที่กำหนดไว้
  // (ใช้ account number เป็นเกณฑ์)
  if FAccountNumber < Target.FAccountNumber then
  begin
    FLock.Acquire;
    try
      Target.FLock.Acquire;
      try
        if FBalance >= Amount then
        begin
          FBalance := FBalance - Amount;
          Target.FBalance := Target.FBalance + Amount;
          WriteLn(Format('[Transfer] %s -> %s: %.2f', 
            [FAccountNumber, Target.FAccountNumber, Amount]));
          Result := True;
        end;
      finally
        Target.FLock.Release;
      end;
    finally
      FLock.Release;
    end;
  end
  else
  begin
    Target.FLock.Acquire;
    try
      FLock.Acquire;
      try
        if FBalance >= Amount then
        begin
          FBalance := FBalance - Amount;
          Target.FBalance := Target.FBalance + Amount;
          WriteLn(Format('[Transfer] %s -> %s: %.2f', 
            [FAccountNumber, Target.FAccountNumber, Amount]));
          Result := True;
        end;
      finally
        FLock.Release;
      end;
    finally
      Target.FLock.Release;
    end;
  end;
end;

function TBankAccount.GetBalance: double;
begin
  FLock.Acquire;
  try
    Result := FBalance;
  finally
    FLock.Release;
  end;
end;

// Test banking threads
type
  TBankingThread = class(TThread)
  private
    FAccount1, FAccount2: TBankAccount;
    FID: integer;
  protected
    procedure Execute; override;
  public
    constructor Create(Acc1, Acc2: TBankAccount; ID: integer);
  end;

constructor TBankingThread.Create(Acc1, Acc2: TBankAccount; ID: integer);
begin
  inherited Create(True);
  FAccount1 := Acc1;
  FAccount2 := Acc2;
  FID := ID;
  FreeOnTerminate := False;
end;

procedure TBankingThread.Execute;
var
  i: integer;
begin
  for i := 1 to 3 do
  begin
    if Terminated then Break;
    
    // สลับ transfer ไปมา
    if FID mod 2 = 0 then
      FAccount1.Transfer(FAccount2, 1000)
    else
      FAccount2.Transfer(FAccount1, 500);
    
    Sleep(100);
  end;
end;

var
  Account1, Account2: TBankAccount;
  Threads: array[0..3] of TBankingThread;
  i: integer;
  TotalBefore, TotalAfter: double;
begin
  WriteLn('=== Banking System with Mutex ===');
  
  Account1 := TBankAccount.Create('ACC-001', 'สมชาย', 50000);
  Account2 := TBankAccount.Create('ACC-002', 'สมหญิง', 30000);
  
  TotalBefore := Account1.GetBalance + Account2.GetBalance;
  WriteLn('Initial balance - ACC1: ', Account1.GetBalance:0:2,
    ' ACC2: ', Account2.GetBalance:0:2);
  WriteLn('Total: ', TotalBefore:0:2);
  WriteLn('');
  
  try
    for i := 0 to 3 do
    begin
      Threads[i] := TBankingThread.Create(Account1, Account2, i);
      Threads[i].Start;
    end;
    
    for i := 0 to 3 do
      Threads[i].WaitFor;
    
    TotalAfter := Account1.GetBalance + Account2.GetBalance;
    WriteLn('');
    WriteLn('Final balance - ACC1: ', Account1.GetBalance:0:2,
      ' ACC2: ', Account2.GetBalance:0:2);
    WriteLn('Total: ', TotalAfter:0:2);
    
    if Abs(TotalBefore - TotalAfter) < 0.01 then
      WriteLn('Money conservation: PASSED!')
    else
      WriteLn('Money conservation: FAILED!');
    
  finally
    for i := 0 to 3 do
      Threads[i].Free;
    Account1.Free;
    Account2.Free;
  end;
end.
```

---

## 43.8 Semaphores

### การใช้ Semaphore

```pascal
program SemaphoreDemo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils
  {$IFDEF WINDOWS}, Windows{$ENDIF}
  {$IFDEF UNIX}, Semaphore{$ENDIF};

type
  // Counting Semaphore
  TCountingSemaphore = class
  private
    FCount: integer;
    FMaxCount: integer;
    FLock: TCriticalSection;
    FAvailEvent: TEvent;
  public
    constructor Create(InitialCount, MaxCount: integer);
    destructor Destroy; override;
    
    procedure Acquire;                  // Wait / P
    function TryAcquire(TimeoutMs: cardinal = 0): boolean;
    procedure Release;                  // Signal / V
    
    property Count: integer read FCount;
    property MaxCount: integer read FMaxCount;
  end;

constructor TCountingSemaphore.Create(InitialCount, MaxCount: integer);
begin
  inherited Create;
  FCount := InitialCount;
  FMaxCount := MaxCount;
  FLock := TCriticalSection.Create;
  FAvailEvent := TEvent.Create(nil, False, InitialCount > 0, '');
end;

destructor TCountingSemaphore.Destroy;
begin
  FAvailEvent.Free;
  FLock.Free;
  inherited Destroy;
end;

procedure TCountingSemaphore.Acquire;
begin
  // รอจนมี resource ว่าง
  FAvailEvent.WaitFor(INFINITE);
  
  FLock.Acquire;
  try
    Dec(FCount);
    // ถ้ายังมีเหลือ signal ต่อ
    if FCount > 0 then
      FAvailEvent.SetEvent;
  finally
    FLock.Release;
  end;
end;

function TCountingSemaphore.TryAcquire(TimeoutMs: cardinal): boolean;
begin
  if FAvailEvent.WaitFor(TimeoutMs) <> wrSignaled then
    Exit(False);
    
  FLock.Acquire;
  try
    if FCount <= 0 then
      Exit(False);
      
    Dec(FCount);
    if FCount > 0 then
      FAvailEvent.SetEvent;
    Result := True;
  finally
    FLock.Release;
  end;
end;

procedure TCountingSemaphore.Release;
begin
  FLock.Acquire;
  try
    if FCount < FMaxCount then
    begin
      Inc(FCount);
      FAvailEvent.SetEvent;
    end;
  finally
    FLock.Release;
  end;
end;

// Connection Pool ที่ใช้ Semaphore
type
  TDBConnection = class
  private
    FID: integer;
    FInUse: boolean;
  public
    constructor Create(ID: integer);
    procedure Connect;
    procedure Disconnect;
    procedure ExecuteQuery(const SQL: string);
    property ID: integer read FID;
    property InUse: boolean read FInUse write FInUse;
  end;
  
  TConnectionPool = class
  private
    FConnections: array of TDBConnection;
    FSemaphore: TCountingSemaphore;
    FLock: TCriticalSection;
    FPoolSize: integer;
    
    function GetFreeConnection: TDBConnection;
  public
    constructor Create(PoolSize: integer);
    destructor Destroy; override;
    
    function AcquireConnection: TDBConnection;
    procedure ReleaseConnection(Conn: TDBConnection);
    
    property PoolSize: integer read FPoolSize;
  end;

constructor TDBConnection.Create(ID: integer);
begin
  inherited Create;
  FID := ID;
  FInUse := False;
end;

procedure TDBConnection.Connect;
begin
  WriteLn(Format('[Conn-%d] Connected', [FID]));
  Sleep(50); // simulate connect time
end;

procedure TDBConnection.Disconnect;
begin
  WriteLn(Format('[Conn-%d] Disconnected', [FID]));
end;

procedure TDBConnection.ExecuteQuery(const SQL: string);
begin
  WriteLn(Format('[Conn-%d] Execute: %s', [FID, SQL]));
  Sleep(200 + Random(300)); // simulate query time
end;

constructor TConnectionPool.Create(PoolSize: integer);
var
  i: integer;
begin
  inherited Create;
  FPoolSize := PoolSize;
  SetLength(FConnections, PoolSize);
  FLock := TCriticalSection.Create;
  FSemaphore := TCountingSemaphore.Create(PoolSize, PoolSize);
  
  for i := 0 to PoolSize - 1 do
  begin
    FConnections[i] := TDBConnection.Create(i + 1);
    FConnections[i].Connect;
  end;
end;

destructor TConnectionPool.Destroy;
var
  i: integer;
begin
  for i := 0 to FPoolSize - 1 do
  begin
    FConnections[i].Disconnect;
    FConnections[i].Free;
  end;
  
  FSemaphore.Free;
  FLock.Free;
  inherited Destroy;
end;

function TConnectionPool.GetFreeConnection: TDBConnection;
var
  i: integer;
begin
  Result := nil;
  FLock.Acquire;
  try
    for i := 0 to FPoolSize - 1 do
      if not FConnections[i].InUse then
      begin
        Result := FConnections[i];
        Result.InUse := True;
        Break;
      end;
  finally
    FLock.Release;
  end;
end;

function TConnectionPool.AcquireConnection: TDBConnection;
begin
  // รอจนมี connection ว่าง
  FSemaphore.Acquire;
  Result := GetFreeConnection;
end;

procedure TConnectionPool.ReleaseConnection(Conn: TDBConnection);
begin
  FLock.Acquire;
  try
    Conn.InUse := False;
  finally
    FLock.Release;
  end;
  FSemaphore.Release;
end;

// Worker threads ที่ใช้ connection pool
type
  TWorkerThread = class(TThread)
  private
    FPool: TConnectionPool;
    FID: integer;
    FQueries: integer;
  protected
    procedure Execute; override;
  public
    constructor Create(Pool: TConnectionPool; ID, Queries: integer);
  end;

constructor TWorkerThread.Create(Pool: TConnectionPool; ID, Queries: integer);
begin
  inherited Create(True);
  FPool := Pool;
  FID := ID;
  FQueries := Queries;
  FreeOnTerminate := False;
end;

procedure TWorkerThread.Execute;
var
  Conn: TDBConnection;
  i: integer;
begin
  for i := 1 to FQueries do
  begin
    if Terminated then Break;
    
    WriteLn(Format('[Worker-%d] Requesting connection for query %d...', [FID, i]));
    
    Conn := FPool.AcquireConnection;
    try
      Conn.ExecuteQuery(Format('SELECT * FROM table%d WHERE id=%d', [FID, i]));
    finally
      FPool.ReleaseConnection(Conn);
      WriteLn(Format('[Worker-%d] Connection released', [FID]));
    end;
  end;
end;

var
  Pool: TConnectionPool;
  Workers: array[0..7] of TWorkerThread;
  i: integer;
begin
  Randomize;
  WriteLn('=== Connection Pool Demo (Semaphore) ===');
  WriteLn('Pool size: 3 connections, 8 workers');
  WriteLn('');
  
  Pool := TConnectionPool.Create(3); // เพียง 3 connections สำหรับ 8 workers
  try
    for i := 0 to 7 do
    begin
      Workers[i] := TWorkerThread.Create(Pool, i + 1, 2);
      Workers[i].Start;
    end;
    
    for i := 0 to 7 do
      Workers[i].WaitFor;
    
    WriteLn('');
    WriteLn('All workers completed!');
  finally
    for i := 0 to 7 do
      Workers[i].Free;
    Pool.Free;
  end;
end.
```

---

## 43.9 Thread Pools

### Thread Pool Implementation

```pascal
program ThreadPool;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Contnrs;

type
  TWorkItem = class
  public
    procedure Execute; virtual; abstract;
  end;
  
  TFunctionWorkItem = class(TWorkItem)
  private
    FProc: TProcedure;
  public
    constructor Create(Proc: TProcedure);
    procedure Execute; override;
  end;
  
  TThreadPool = class;
  
  TPoolWorker = class(TThread)
  private
    FPool: TThreadPool;
    FIsIdle: boolean;
  protected
    procedure Execute; override;
  public
    constructor Create(Pool: TThreadPool);
    property IsIdle: boolean read FIsIdle;
  end;
  
  TThreadPool = class
  private
    FWorkers: TList;
    FWorkQueue: TObjectQueue;
    FQueueLock: TCriticalSection;
    FWorkAvailable: TEvent;
    FMinWorkers: integer;
    FMaxWorkers: integer;
    FActiveWork: integer;
    FShutdown: boolean;
    
    function DequeueWork: TWorkItem;
    procedure WorkCompleted;
  public
    constructor Create(MinWorkers, MaxWorkers: integer);
    destructor Destroy; override;
    
    procedure Submit(WorkItem: TWorkItem);
    procedure SubmitProc(Proc: TProcedure);
    procedure WaitAll;
    procedure Shutdown;
    
    function GetWorkerCount: integer;
    function GetQueueSize: integer;
    function GetActiveCount: integer;
  end;

constructor TFunctionWorkItem.Create(Proc: TProcedure);
begin
  inherited Create;
  FProc := Proc;
end;

procedure TFunctionWorkItem.Execute;
begin
  if Assigned(FProc) then
    FProc;
end;

constructor TPoolWorker.Create(Pool: TThreadPool);
begin
  inherited Create(True);
  FPool := Pool;
  FIsIdle := True;
  FreeOnTerminate := False;
end;

procedure TPoolWorker.Execute;
var
  Work: TWorkItem;
begin
  while not Terminated do
  begin
    // รอ work
    FPool.FWorkAvailable.WaitFor(1000);
    
    if Terminated then Break;
    
    // รับ work
    Work := FPool.DequeueWork;
    
    if Assigned(Work) then
    begin
      FIsIdle := False;
      try
        Work.Execute;
        Work.Free;
      except
        on E: Exception do
          WriteLn('[Worker Error] ', E.Message);
      end;
      FIsIdle := True;
      FPool.WorkCompleted;
    end;
  end;
end;

constructor TThreadPool.Create(MinWorkers, MaxWorkers: integer);
var
  i: integer;
  Worker: TPoolWorker;
begin
  inherited Create;
  FMinWorkers := MinWorkers;
  FMaxWorkers := MaxWorkers;
  FActiveWork := 0;
  FShutdown := False;
  
  FWorkers := TList.Create;
  FWorkQueue := TObjectQueue.Create;
  FQueueLock := TCriticalSection.Create;
  FWorkAvailable := TEvent.Create(nil, False, False, '');
  
  // สร้าง minimum workers
  for i := 1 to MinWorkers do
  begin
    Worker := TPoolWorker.Create(Self);
    FWorkers.Add(Worker);
    Worker.Start;
  end;
end;

destructor TThreadPool.Destroy;
begin
  Shutdown;
  
  FWorkAvailable.Free;
  FQueueLock.Free;
  FWorkQueue.Free;
  FWorkers.Free;
  inherited Destroy;
end;

function TThreadPool.DequeueWork: TWorkItem;
begin
  FQueueLock.Acquire;
  try
    if FWorkQueue.Count > 0 then
      Result := TWorkItem(FWorkQueue.Pop)
    else
    begin
      Result := nil;
      FWorkAvailable.ResetEvent;
    end;
  finally
    FQueueLock.Release;
  end;
end;

procedure TThreadPool.WorkCompleted;
begin
  FQueueLock.Acquire;
  try
    Dec(FActiveWork);
  finally
    FQueueLock.Release;
  end;
end;

procedure TThreadPool.Submit(WorkItem: TWorkItem);
begin
  if FShutdown then
    raise Exception.Create('Thread pool is shutdown');
    
  FQueueLock.Acquire;
  try
    FWorkQueue.Push(WorkItem);
    Inc(FActiveWork);
  finally
    FQueueLock.Release;
  end;
  
  FWorkAvailable.SetEvent;
end;

procedure TThreadPool.SubmitProc(Proc: TProcedure);
begin
  Submit(TFunctionWorkItem.Create(Proc));
end;

procedure TThreadPool.WaitAll;
begin
  while True do
  begin
    FQueueLock.Acquire;
    try
      if (FWorkQueue.Count = 0) and (FActiveWork = 0) then
        Break;
    finally
      FQueueLock.Release;
    end;
    Sleep(10);
  end;
end;

procedure TThreadPool.Shutdown;
var
  i: integer;
  Worker: TPoolWorker;
begin
  FShutdown := True;
  
  // สัญญาณให้ workers หยุด
  for i := 0 to FWorkers.Count - 1 do
  begin
    Worker := TPoolWorker(FWorkers[i]);
    Worker.Terminate;
  end;
  
  FWorkAvailable.SetEvent;
  
  // รอ workers จบ
  for i := 0 to FWorkers.Count - 1 do
  begin
    Worker := TPoolWorker(FWorkers[i]);
    Worker.WaitFor;
    Worker.Free;
  end;
  
  FWorkers.Clear;
end;

function TThreadPool.GetWorkerCount: integer;
begin
  Result := FWorkers.Count;
end;

function TThreadPool.GetQueueSize: integer;
begin
  FQueueLock.Acquire;
  try
    Result := FWorkQueue.Count;
  finally
    FQueueLock.Release;
  end;
end;

function TThreadPool.GetActiveCount: integer;
begin
  FQueueLock.Acquire;
  try
    Result := FActiveWork;
  finally
    FQueueLock.Release;
  end;
end;

// Test tasks
var
  TaskCounter: integer;
  TaskLock: TCriticalSection;

procedure TaskA;
begin
  TaskLock.Acquire;
  try
    Inc(TaskCounter);
    WriteLn('[Task A-', TaskCounter, '] Running on thread ', GetCurrentThreadId);
  finally
    TaskLock.Release;
  end;
  Sleep(100 + Random(200));
end;

procedure TaskB;
var
  N: integer;
  i: integer;
begin
  TaskLock.Acquire;
  try
    N := TaskCounter;
    Inc(TaskCounter);
    WriteLn('[Task B-', N, '] Computing...');
  finally
    TaskLock.Release;
  end;
  
  // เลียนแบบงานคำนวณ
  var sum: int64 := 0;
  for i := 1 to 1000000 do
    sum := sum + i;
  
  WriteLn('[Task B-', N, '] Done, sum = ', sum);
end;

var
  Pool: TThreadPool;
  i: integer;
begin
  Randomize;
  TaskCounter := 0;
  TaskLock := TCriticalSection.Create;
  
  WriteLn('=== Thread Pool Demo ===');
  WriteLn('');
  
  Pool := TThreadPool.Create(3, 6);
  try
    WriteLn('Worker count: ', Pool.GetWorkerCount);
    WriteLn('Submitting 10 tasks...');
    WriteLn('');
    
    // ส่ง tasks เข้า pool
    for i := 1 to 5 do
      Pool.SubmitProc(@TaskA);
    for i := 1 to 5 do
      Pool.SubmitProc(@TaskB);
    
    // รอทุก task จบ
    Pool.WaitAll;
    
    WriteLn('');
    WriteLn('All tasks completed!');
    WriteLn('Total tasks processed: ', TaskCounter);
    
  finally
    Pool.Free;
    TaskLock.Free;
  end;
end.
```

---

## 43.10 Complete Example: Background Downloader

```pascal
program BackgroundDownloader;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, fphttpclient;

type
  TDownloadStatus = (dsIdle, dsDownloading, dsPaused, dsCompleted, dsError, dsCancelled);
  
  TDownloadProgress = record
    URL: string;
    FileName: string;
    BytesDownloaded: int64;
    TotalBytes: int64;
    Speed: double; // bytes per second
    Status: TDownloadStatus;
    ErrorMessage: string;
  end;
  
  TProgressEvent = procedure(const Progress: TDownloadProgress) of object;
  TCompleteEvent = procedure(const FileName: string; Success: boolean; const Error: string) of object;

  TDownloadThread = class(TThread)
  private
    FURL: string;
    FOutputPath: string;
    FProgress: TDownloadProgress;
    FProgressLock: TCriticalSection;
    FOnProgress: TProgressEvent;
    FOnComplete: TCompleteEvent;
    FHTTPClient: TFPHTTPClient;
    FPaused: boolean;
    FPauseLock: TCriticalSection;
    
    procedure UpdateProgress(Downloaded, Total: int64);
    function GetProgress: TDownloadProgress;
  protected
    procedure Execute; override;
  public
    constructor Create(const URL, OutputPath: string;
      OnProgress: TProgressEvent; OnComplete: TCompleteEvent);
    destructor Destroy; override;
    
    procedure Pause;
    procedure Resume;
    procedure Cancel;
    
    function GetStatus: TDownloadStatus;
  end;

constructor TDownloadThread.Create(const URL, OutputPath: string;
  OnProgress: TProgressEvent; OnComplete: TCompleteEvent);
begin
  inherited Create(True);
  FURL := URL;
  FOutputPath := OutputPath;
  FOnProgress := OnProgress;
  FOnComplete := OnComplete;
  FPaused := False;
  FreeOnTerminate := False;
  
  FProgressLock := TCriticalSection.Create;
  FPauseLock := TCriticalSection.Create;
  
  FProgress.URL := URL;
  FProgress.FileName := ExtractFileName(OutputPath);
  FProgress.Status := dsIdle;
  FProgress.BytesDownloaded := 0;
  FProgress.TotalBytes := 0;
  FProgress.Speed := 0;
end;

destructor TDownloadThread.Destroy;
begin
  if Assigned(FHTTPClient) then
    FHTTPClient.Free;
  FProgressLock.Free;
  FPauseLock.Free;
  inherited Destroy;
end;

procedure TDownloadThread.UpdateProgress(Downloaded, Total: int64);
begin
  FProgressLock.Acquire;
  try
    FProgress.BytesDownloaded := Downloaded;
    FProgress.TotalBytes := Total;
  finally
    FProgressLock.Release;
  end;
  
  if Assigned(FOnProgress) then
    Synchronize(procedure
    begin
      var P := GetProgress;
      FOnProgress(P);
    end);
end;

function TDownloadThread.GetProgress: TDownloadProgress;
begin
  FProgressLock.Acquire;
  try
    Result := FProgress;
  finally
    FProgressLock.Release;
  end;
end;

procedure TDownloadThread.Execute;
var
  Stream: TFileStream;
  StartTime: TDateTime;
  BytesBefore: int64;
  Elapsed: double;
begin
  FProgressLock.Acquire;
  try
    FProgress.Status := dsDownloading;
  finally
    FProgressLock.Release;
  end;
  
  Stream := nil;
  FHTTPClient := TFPHTTPClient.Create(nil);
  
  try
    try
      StartTime := Now;
      BytesBefore := 0;
      
      Stream := TFileStream.Create(FOutputPath, fmCreate);
      
      FHTTPClient.Get(FURL, Stream);
      
      // คำนวณ speed
      Elapsed := (Now - StartTime) * 86400; // convert to seconds
      if Elapsed > 0 then
      begin
        FProgressLock.Acquire;
        try
          FProgress.Speed := Stream.Size / Elapsed;
          FProgress.BytesDownloaded := Stream.Size;
          FProgress.Status := dsCompleted;
        finally
          FProgressLock.Release;
        end;
      end;
      
      if Assigned(FOnComplete) then
        Synchronize(procedure
        begin
          FOnComplete(FOutputPath, True, '');
        end);
    except
      on E: Exception do
      begin
        FProgressLock.Acquire;
        try
          FProgress.Status := dsError;
          FProgress.ErrorMessage := E.Message;
        finally
          FProgressLock.Release;
        end;
        
        if Assigned(FOnComplete) then
          Synchronize(procedure
          begin
            FOnComplete(FOutputPath, False, E.Message);
          end);
      end;
    end;
  finally
    if Assigned(Stream) then
      Stream.Free;
    FreeAndNil(FHTTPClient);
  end;
end;

procedure TDownloadThread.Pause;
begin
  FPauseLock.Acquire;
  try
    FPaused := True;
    FProgressLock.Acquire;
    try
      FProgress.Status := dsPaused;
    finally
      FProgressLock.Release;
    end;
  finally
    FPauseLock.Release;
  end;
end;

procedure TDownloadThread.Resume;
begin
  FPauseLock.Acquire;
  try
    FPaused := False;
    FProgressLock.Acquire;
    try
      FProgress.Status := dsDownloading;
    finally
      FProgressLock.Release;
    end;
  finally
    FPauseLock.Release;
  end;
end;

procedure TDownloadThread.Cancel;
begin
  Terminate;
  if Assigned(FHTTPClient) then
  begin
    FProgressLock.Acquire;
    try
      FProgress.Status := dsCancelled;
    finally
      FProgressLock.Release;
    end;
  end;
end;

function TDownloadThread.GetStatus: TDownloadStatus;
begin
  FProgressLock.Acquire;
  try
    Result := FProgress.Status;
  finally
    FProgressLock.Release;
  end;
end;

// Download Manager
type
  TDownloadManager = class
  private
    FThreads: TList;
    FLock: TCriticalSection;
    
    procedure OnProgress(const Progress: TDownloadProgress);
    procedure OnComplete(const FileName: string; Success: boolean; const Error: string);
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Download(const URL, OutputPath: string);
    procedure WaitAll;
    procedure CancelAll;
    
    function GetActiveCount: integer;
  end;

constructor TDownloadManager.Create;
begin
  inherited Create;
  FThreads := TList.Create;
  FLock := TCriticalSection.Create;
end;

destructor TDownloadManager.Destroy;
begin
  CancelAll;
  FLock.Free;
  FThreads.Free;
  inherited Destroy;
end;

procedure TDownloadManager.OnProgress(const Progress: TDownloadProgress);
var
  Pct: double;
begin
  if Progress.TotalBytes > 0 then
    Pct := Progress.BytesDownloaded / Progress.TotalBytes * 100
  else
    Pct := 0;
    
  Write(Format(#13'[%s] %.1f%% (%.1f KB/s)', [
    Progress.FileName,
    Pct,
    Progress.Speed / 1024
  ]));
end;

procedure TDownloadManager.OnComplete(const FileName: string; 
  Success: boolean; const Error: string);
begin
  WriteLn;
  if Success then
    WriteLn('[Complete] ', FileName)
  else
    WriteLn('[Error] ', FileName, ': ', Error);
end;

procedure TDownloadManager.Download(const URL, OutputPath: string);
var
  Thread: TDownloadThread;
begin
  Thread := TDownloadThread.Create(URL, OutputPath, @OnProgress, @OnComplete);
  
  FLock.Acquire;
  try
    FThreads.Add(Thread);
  finally
    FLock.Release;
  end;
  
  Thread.Start;
end;

procedure TDownloadManager.WaitAll;
var
  i: integer;
  Thread: TDownloadThread;
begin
  FLock.Acquire;
  try
    for i := 0 to FThreads.Count - 1 do
    begin
      Thread := TDownloadThread(FThreads[i]);
      FLock.Release;
      Thread.WaitFor;
      FLock.Acquire;
    end;
  finally
    FLock.Release;
  end;
end;

procedure TDownloadManager.CancelAll;
var
  i: integer;
  Thread: TDownloadThread;
begin
  FLock.Acquire;
  try
    for i := 0 to FThreads.Count - 1 do
    begin
      Thread := TDownloadThread(FThreads[i]);
      Thread.Cancel;
    end;
  finally
    FLock.Release;
  end;
  
  WaitAll;
  
  FLock.Acquire;
  try
    for i := 0 to FThreads.Count - 1 do
      TDownloadThread(FThreads[i]).Free;
    FThreads.Clear;
  finally
    FLock.Release;
  end;
end;

function TDownloadManager.GetActiveCount: integer;
var
  i: integer;
begin
  Result := 0;
  FLock.Acquire;
  try
    for i := 0 to FThreads.Count - 1 do
      if not TDownloadThread(FThreads[i]).Finished then
        Inc(Result);
  finally
    FLock.Release;
  end;
end;

begin
  WriteLn('=== Background Downloader Demo ===');
  WriteLn('(This requires actual URLs to download from)');
  WriteLn('');
  
  // Demo: แสดงโครงสร้างเท่านั้น
  var Manager := TDownloadManager.Create;
  try
    WriteLn('Download Manager created');
    WriteLn('Ready to download files...');
    
    // ถ้าต้องการ download จริง:
    // Manager.Download('https://example.com/file.zip', 'output.zip');
    // Manager.WaitAll;
    
    WriteLn('Demo complete!');
  finally
    Manager.Free;
  end;
end.
```

---

## 43.11 Complete Example: Parallel Processing

```pascal
program ParallelProcessing;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Math;

type
  TDataChunk = record
    StartIndex: integer;
    EndIndex: integer;
    Data: array of double;
    Sum: double;
    Max: double;
    Min: double;
    Mean: double;
    StdDev: double;
  end;
  PDatabaseChunk = ^TDataChunk;

  TStatThread = class(TThread)
  private
    FChunk: PDatabaseChunk;
  protected
    procedure Execute; override;
  public
    constructor Create(Chunk: PDatabaseChunk);
  end;

constructor TStatThread.Create(Chunk: PDatabaseChunk);
begin
  inherited Create(True);
  FChunk := Chunk;
  FreeOnTerminate := False;
end;

procedure TStatThread.Execute;
var
  i: integer;
  SumSq: double;
  N: integer;
begin
  if FChunk = nil then Exit;
  
  N := FChunk^.EndIndex - FChunk^.StartIndex + 1;
  if N <= 0 then Exit;
  
  // คำนวณ sum, max, min
  FChunk^.Sum := 0;
  FChunk^.Max := FChunk^.Data[0];
  FChunk^.Min := FChunk^.Data[0];
  
  for i := 0 to N - 1 do
  begin
    FChunk^.Sum := FChunk^.Sum + FChunk^.Data[i];
    if FChunk^.Data[i] > FChunk^.Max then FChunk^.Max := FChunk^.Data[i];
    if FChunk^.Data[i] < FChunk^.Min then FChunk^.Min := FChunk^.Data[i];
  end;
  
  FChunk^.Mean := FChunk^.Sum / N;
  
  // คำนวณ standard deviation
  SumSq := 0;
  for i := 0 to N - 1 do
    SumSq := SumSq + Sqr(FChunk^.Data[i] - FChunk^.Mean);
  
  FChunk^.StdDev := Sqrt(SumSq / N);
end;

procedure ParallelStats(const Data: array of double; NumThreads: integer);
var
  Threads: array of TStatThread;
  Chunks: array of TDataChunk;
  ChunkSize: integer;
  i, j: integer;
  TotalN: integer;
  GlobalSum, GlobalMax, GlobalMin: double;
  GlobalMean, GlobalStdDev: double;
  StartTime, EndTime: TDateTime;
begin
  TotalN := Length(Data);
  if TotalN = 0 then Exit;
  
  WriteLn('Data size: ', TotalN, ' values');
  WriteLn('Threads: ', NumThreads);
  WriteLn('');
  
  SetLength(Threads, NumThreads);
  SetLength(Chunks, NumThreads);
  
  ChunkSize := (TotalN + NumThreads - 1) div NumThreads;
  
  StartTime := Now;
  
  // แบ่งข้อมูลเป็น chunks
  for i := 0 to NumThreads - 1 do
  begin
    Chunks[i].StartIndex := i * ChunkSize;
    Chunks[i].EndIndex := Min((i + 1) * ChunkSize - 1, TotalN - 1);
    
    var ChunkN := Chunks[i].EndIndex - Chunks[i].StartIndex + 1;
    SetLength(Chunks[i].Data, ChunkN);
    
    for j := 0 to ChunkN - 1 do
      Chunks[i].Data[j] := Data[Chunks[i].StartIndex + j];
    
    Threads[i] := TStatThread.Create(@Chunks[i]);
    Threads[i].Start;
  end;
  
  // รอทุก thread จบ
  for i := 0 to NumThreads - 1 do
    Threads[i].WaitFor;
  
  EndTime := Now;
  
  // รวมผลลัพธ์
  GlobalSum := 0;
  GlobalMax := Chunks[0].Max;
  GlobalMin := Chunks[0].Min;
  
  for i := 0 to NumThreads - 1 do
  begin
    GlobalSum := GlobalSum + Chunks[i].Sum;
    if Chunks[i].Max > GlobalMax then GlobalMax := Chunks[i].Max;
    if Chunks[i].Min < GlobalMin then GlobalMin := Chunks[i].Min;
  end;
  
  GlobalMean := GlobalSum / TotalN;
  
  // คำนวณ global std dev
  var SumSq: double := 0;
  for i := 0 to TotalN - 1 do
    SumSq := SumSq + Sqr(Data[i] - GlobalMean);
  GlobalStdDev := Sqrt(SumSq / TotalN);
  
  // Cleanup
  for i := 0 to NumThreads - 1 do
    Threads[i].Free;
  
  WriteLn('=== Statistics Results ===');
  WriteLn(Format('Count:   %d', [TotalN]));
  WriteLn(Format('Sum:     %.4f', [GlobalSum]));
  WriteLn(Format('Mean:    %.4f', [GlobalMean]));
  WriteLn(Format('StdDev:  %.4f', [GlobalStdDev]));
  WriteLn(Format('Min:     %.4f', [GlobalMin]));
  WriteLn(Format('Max:     %.4f', [GlobalMax]));
  WriteLn('');
  WriteLn('Time: ', FormatDateTime('s.zzz', EndTime - StartTime), ' seconds');
end;

var
  Data: array of double;
  N: integer;
  i: integer;
begin
  Randomize;
  N := 10000;
  
  WriteLn('=== Parallel Statistics Processing ===');
  WriteLn('');
  
  // สร้างข้อมูลทดสอบ
  SetLength(Data, N);
  for i := 0 to N - 1 do
    Data[i] := Random * 1000 - 500; // -500 ถึง 500
  
  WriteLn('Using 4 threads:');
  ParallelStats(Data, 4);
  WriteLn('');
  WriteLn('Using 1 thread (for comparison):');
  ParallelStats(Data, 1);
end.
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: Producer-Consumer Pattern
สร้าง producer-consumer ที่:
- Producer thread สร้าง tasks
- Consumer threads ประมวลผล
- ใช้ blocking queue ระหว่างกัน

### ข้อ 2: Read-Write Lock
สร้าง RWLock (Multiple readers, single writer):
- อนุญาต readers หลายคนพร้อมกัน
- Writer ต้องรอ readers ทั้งหมดออกก่อน
- ทดสอบด้วย concurrent access

### ข้อ 3: Thread-Safe Cache
สร้าง LRU cache ที่ thread-safe:
- รองรับ concurrent reads
- Thread-safe writes
- Automatic eviction

### ข้อ 4: Parallel File Processing
ประมวลผลไฟล์ขนาดใหญ่แบบ parallel:
- แบ่งไฟล์เป็น chunks
- ประมวลผลแต่ละ chunk ด้วย thread
- รวมผลลัพธ์

### ข้อ 5: Event Bus
สร้าง thread-safe event bus:
- Subscribe/Unsubscribe handlers
- Publish events จาก any thread
- Handle delivery ใน main thread

### ข้อ 6: Parallel Sort
สร้าง parallel merge sort:
- แบ่ง array เป็นส่วนย่อย
- Sort แต่ละส่วนด้วย thread
- Merge ผลลัพธ์

### ข้อ 7: Web Crawler
Web crawler แบบ multi-threaded:
- Thread pool สำหรับ fetching
- Queue สำหรับ URLs
- Report results

### ข้อ 8: Scheduler
Job scheduler:
- Schedule tasks ตาม time
- Recurring tasks
- Priority queue

### ข้อ 9: Progress Monitor
Monitor สำหรับติดตาม multiple tasks:
- แสดง progress ของแต่ละ task
- รวม overall progress
- Handle errors

### ข้อ 10: Pipeline Processing
สร้าง processing pipeline:
- Stage 1: อ่านข้อมูล
- Stage 2: transform
- Stage 3: validate
- Stage 4: บันทึก
- แต่ละ stage ทำงาน parallel
