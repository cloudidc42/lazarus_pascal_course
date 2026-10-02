# Part 60 - Memory Management ใน Lazarus/Pascal

## บทนำ

Memory Management คือหัวใจสำคัญของการเขียนโปรแกรมที่มีประสิทธิภาพ Pascal/Lazarus ให้ควบคุม Memory ได้โดยตรง ซึ่งทั้งเป็นข้อดีและต้องระมัดระวัง

---

## 1. Stack vs Heap

### Stack Memory

```pascal
program StackVsHeap;

{$mode objfpc}{$H+}

uses SysUtils;

{ Stack: ตัวแปรท้องถิ่น, Parameters, Local Records }
procedure StackDemo;
var
  { ตัวแปรเหล่านี้อยู่ใน Stack }
  X: Integer;          { 4 bytes }
  Y: Double;           { 8 bytes }
  Name: string[50];    { Short string: อยู่ใน Stack }
  Buffer: array[0..99] of Byte;  { 100 bytes ใน Stack }
  
  { Record ก็อยู่ใน Stack }
  Point: record
    X, Y: Double;
  end;
begin
  X := 42;
  Y := 3.14;
  Name := 'Hello';
  
  { Stack จะถูก Pop อัตโนมัติเมื่อออกจาก Procedure }
  WriteLn('X = ', X);
  WriteLn('Address of X: ', HexStr(@X, 8));
  WriteLn('Address of Y: ', HexStr(@Y, 8));
  { สังเกต: Address ใกล้กัน (อยู่ใน Stack Frame เดียว) }
end;

{ Heap: Dynamic Allocation }
procedure HeapDemo;
var
  { Pointer ชี้ไปยัง Heap }
  P: ^Integer;
  S: string;              { Long string: อยู่ใน Heap }
  Arr: array of Integer;  { Dynamic array: อยู่ใน Heap }
  Obj: TObject;           { Object: อยู่ใน Heap }
begin
  { Allocate ใน Heap }
  New(P);
  try
    P^ := 42;
    WriteLn('Heap Integer: ', P^);
    WriteLn('Address: ', HexStr(P, 8));
  finally
    Dispose(P);  { ต้อง Free เอง! }
  end;
  
  { Dynamic Array อยู่ใน Heap }
  SetLength(Arr, 100);
  try
    Arr[0] := 1;
    WriteLn('Heap Array[0]: ', Arr[0]);
  finally
    SetLength(Arr, 0);  { Free อัตโนมัติเมื่อ Length = 0 หรือออก Scope }
  end;
  
  { String อยู่ใน Heap (Reference Counting) }
  S := 'Hello, World!';
  WriteLn('Heap String: ', S);
  { Free อัตโนมัติเมื่อออก Scope }
end;

{ เปรียบเทียบขนาด }
procedure SizeDemo;
type
  TSmallRec = record
    A: Byte;
    B: Integer;   { อาจมี Padding }
    C: Byte;
  end;
  
  TPackedRec = packed record
    A: Byte;
    B: Integer;
    C: Byte;
  end;
begin
  WriteLn('SizeOf(TSmallRec)  = ', SizeOf(TSmallRec));   { อาจเป็น 12 (Padding) }
  WriteLn('SizeOf(TPackedRec) = ', SizeOf(TPackedRec));  { 6 (ไม่มี Padding) }
  WriteLn('SizeOf(Integer)    = ', SizeOf(Integer));
  WriteLn('SizeOf(Int64)      = ', SizeOf(Int64));
  WriteLn('SizeOf(Double)     = ', SizeOf(Double));
  WriteLn('SizeOf(Pointer)    = ', SizeOf(Pointer));
  WriteLn('SizeOf(TObject)    = ', SizeOf(TObject));  { แค่ Pointer }
end;

begin
  WriteLn('=== Stack Demo ===');
  StackDemo;
  
  WriteLn(#13#10 + '=== Heap Demo ===');
  HeapDemo;
  
  WriteLn(#13#10 + '=== Size Demo ===');
  SizeDemo;
end.
```

---

## 2. Manual Memory Management

```pascal
unit ManualMemory;

{$mode objfpc}{$H+}

interface

uses SysUtils;

type
  { ตัวอย่าง Custom Allocator }
  TMemoryTracker = class
  private
    FAllocCount: Int64;
    FFreeCount: Int64;
    FTotalAllocated: Int64;
    FCurrentUsage: Int64;
    FPeakUsage: Int64;
  public
    procedure TrackAlloc(ASize: NativeUInt);
    procedure TrackFree(ASize: NativeUInt);
    procedure PrintStats;
    
    property AllocCount: Int64 read FAllocCount;
    property FreeCount: Int64 read FFreeCount;
    property CurrentUsage: Int64 read FCurrentUsage;
    property PeakUsage: Int64 read FPeakUsage;
  end;

{ เครื่องมือ Helper สำหรับ Memory }
procedure SafeFree(var AObject);
procedure SafeDispose(var APointer);
function AllocMem(ASize: NativeUInt): Pointer;
procedure FreeMem(APointer: Pointer; ASize: NativeUInt = 0);

implementation

{ TMemoryTracker }
procedure TMemoryTracker.TrackAlloc(ASize: NativeUInt);
begin
  Inc(FAllocCount);
  Inc(FTotalAllocated, ASize);
  Inc(FCurrentUsage, ASize);
  if FCurrentUsage > FPeakUsage then
    FPeakUsage := FCurrentUsage;
end;

procedure TMemoryTracker.TrackFree(ASize: NativeUInt);
begin
  Inc(FFreeCount);
  Dec(FCurrentUsage, ASize);
end;

procedure TMemoryTracker.PrintStats;
begin
  WriteLn('=== Memory Stats ===');
  WriteLn('Allocations:    ', FAllocCount);
  WriteLn('Frees:          ', FFreeCount);
  WriteLn('Leaked allocs:  ', FAllocCount - FFreeCount);
  WriteLn('Total allocated:', FTotalAllocated, ' bytes');
  WriteLn('Current usage:  ', FCurrentUsage, ' bytes');
  WriteLn('Peak usage:     ', FPeakUsage, ' bytes');
end;

{ Helper Functions }
procedure SafeFree(var AObject);
var
  Obj: TObject;
begin
  Obj := TObject(AObject);
  if Obj <> nil then
  begin
    TObject(AObject) := nil;  { Set nil ก่อน Free เพื่อป้องกัน Double-Free }
    Obj.Free;
  end;
end;

procedure SafeDispose(var APointer);
begin
  if Pointer(APointer) <> nil then
  begin
    System.FreeMem(Pointer(APointer));
    Pointer(APointer) := nil;
  end;
end;

function AllocMem(ASize: NativeUInt): Pointer;
begin
  Result := System.GetMem(ASize);
  if Result = nil then
    raise EOutOfMemory.Create('ไม่สามารถ Allocate Memory ได้');
  FillChar(Result^, ASize, 0);  { Initialize เป็น 0 }
end;

procedure FreeMem(APointer: Pointer; ASize: NativeUInt);
begin
  if APointer <> nil then
    System.FreeMem(APointer);
end;

end.
```

---

## 3. Reference Counting

```pascal
unit RefCounting;

{$mode objfpc}{$H+}

interface

uses SysUtils, SyncObjs;

type
  { Interface สำหรับ Reference Counting }
  IRefCounted = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    function AddRef: Integer;
    function Release: Integer;
    function GetRefCount: Integer;
    property RefCount: Integer read GetRefCount;
  end;

  { Base class สำหรับ Reference Counted Objects }
  TRefCountedObject = class(TInterfacedObject)
  private
    { TInterfacedObject จัดการ Reference Counting ให้อัตโนมัติ }
  public
    procedure AfterConstruction; override;
    procedure BeforeDestruction; override;
    
    class function NewInstance: TObject; override;
  end;

  { Smart Pointer Pattern }
  generic TSmartPtr<T: class> = class
  private
    FObj: T;
    FOwned: Boolean;
  public
    constructor Create(AObj: T; AOwned: Boolean = True);
    destructor Destroy; override;
    
    function Get: T; inline;
    procedure Release;
    
    property Value: T read FObj;
  end;

  { Shared Pointer - หลาย Owner }
  generic TSharedPtr<T: class> = class
  private type
    TControlBlock = class
      Obj: T;
      RefCount: Integer;
      Lock: TCriticalSection;
      constructor Create(AObj: T);
      destructor Destroy; override;
    end;
    
  private
    FControl: TControlBlock;
    
  public
    constructor Create(AObj: T = nil);
    constructor CreateShared(ASource: specialize TSharedPtr<T>);
    destructor Destroy; override;
    
    function Get: T;
    function IsValid: Boolean;
    function UseCount: Integer;
    
    property Value: T read Get;
  end;

implementation

{ TRefCountedObject }
procedure TRefCountedObject.AfterConstruction;
begin
  { ลด RefCount จาก 1 กลับเป็น 0 หลัง Constructor }
  inherited;
end;

procedure TRefCountedObject.BeforeDestruction;
begin
  inherited;
end;

class function TRefCountedObject.NewInstance: TObject;
begin
  Result := inherited NewInstance;
end;

{ TSmartPtr }
constructor specialize TSmartPtr<T>.Create(AObj: T; AOwned: Boolean);
begin
  FObj := AObj;
  FOwned := AOwned;
end;

destructor specialize TSmartPtr<T>.Destroy;
begin
  if FOwned and (FObj <> nil) then
    FObj.Free;
  inherited;
end;

function specialize TSmartPtr<T>.Get: T;
begin
  Result := FObj;
end;

procedure specialize TSmartPtr<T>.Release;
begin
  FObj := nil;
  FOwned := False;
end;

{ TSharedPtr.TControlBlock }
constructor specialize TSharedPtr<T>.TControlBlock.Create(AObj: T);
begin
  Obj := AObj;
  RefCount := 1;
  Lock := TCriticalSection.Create;
end;

destructor specialize TSharedPtr<T>.TControlBlock.Destroy;
begin
  Obj.Free;
  Lock.Free;
  inherited;
end;

{ TSharedPtr }
constructor specialize TSharedPtr<T>.Create(AObj: T);
begin
  if AObj <> nil then
    FControl := TControlBlock.Create(AObj);
end;

constructor specialize TSharedPtr<T>.CreateShared(ASource: specialize TSharedPtr<T>);
begin
  FControl := ASource.FControl;
  if FControl <> nil then
  begin
    FControl.Lock.Enter;
    try
      Inc(FControl.RefCount);
    finally
      FControl.Lock.Leave;
    end;
  end;
end;

destructor specialize TSharedPtr<T>.Destroy;
begin
  if FControl <> nil then
  begin
    FControl.Lock.Enter;
    try
      Dec(FControl.RefCount);
      if FControl.RefCount = 0 then
      begin
        FControl.Lock.Leave;
        FControl.Free;
        FControl := nil;
        Exit;
      end;
    finally
      if FControl <> nil then
        FControl.Lock.Leave;
    end;
  end;
  inherited;
end;

function specialize TSharedPtr<T>.Get: T;
begin
  if FControl <> nil then
    Result := FControl.Obj
  else
    Result := nil;
end;

function specialize TSharedPtr<T>.IsValid: Boolean;
begin
  Result := (FControl <> nil) and (FControl.Obj <> nil);
end;

function specialize TSharedPtr<T>.UseCount: Integer;
begin
  if FControl <> nil then
    Result := FControl.RefCount
  else
    Result := 0;
end;

end.
```

---

## 4. Memory Pool

```pascal
unit MemoryPool;

{$mode objfpc}{$H+}

interface

uses SysUtils, SyncObjs;

type
  { Fixed-Size Block Pool }
  TFixedPool = class
  private type
    PBlock = ^TBlock;
    TBlock = record
      Next: PBlock;     { Free list link }
      Data: record end; { Flexible data }
    end;
    
  private
    FBlockSize: NativeUInt;
    FFreeList: PBlock;
    FChunks: array of Pointer;
    FChunkSize: Integer;
    FBlocksPerChunk: Integer;
    FAllocCount: Int64;
    FFreeCount: Int64;
    FLock: TCriticalSection;
    
    procedure AllocateChunk;
    
  public
    constructor Create(ABlockSize: NativeUInt; ABlocksPerChunk: Integer = 256);
    destructor Destroy; override;
    
    function Alloc: Pointer;
    procedure Free(APtr: Pointer);
    
    procedure PrintStats;
    
    property AllocCount: Int64 read FAllocCount;
    property FreeCount: Int64 read FFreeCount;
    property LiveCount: Int64 read GetLiveCount;
    function GetLiveCount: Int64;
  end;

  { Variable-Size Slab Allocator }
  TSlabAllocator = class
  private type
    TSlabClass = record
      Pool: TFixedPool;
      BlockSize: Integer;
    end;
    
  private
    FSlabs: array of TSlabClass;
    FSlabCount: Integer;
    
    function FindSlab(ASize: Integer): Integer;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    function Alloc(ASize: Integer): Pointer;
    procedure Free(APtr: Pointer; ASize: Integer);
  end;

  { Stack-based Arena Allocator }
  TArenaAllocator = class
  private
    FMemory: Pointer;
    FCapacity: NativeUInt;
    FOffset: NativeUInt;
    FMarks: array of NativeUInt;
    FMarkCount: Integer;
    
  public
    constructor Create(ACapacity: NativeUInt);
    destructor Destroy; override;
    
    function Alloc(ASize: NativeUInt; AAlignment: NativeUInt = 8): Pointer;
    procedure Mark;
    procedure PopMark;
    procedure Reset;
    
    property Used: NativeUInt read FOffset;
    property Capacity: NativeUInt read FCapacity;
    property Available: NativeUInt read GetAvailable;
    function GetAvailable: NativeUInt;
  end;

implementation

{ TFixedPool }
constructor TFixedPool.Create(ABlockSize: NativeUInt; ABlocksPerChunk: Integer);
begin
  { Block ต้องใหญ่พอสำหรับ Free List Pointer }
  if ABlockSize < SizeOf(Pointer) then
    FBlockSize := SizeOf(Pointer)
  else
    FBlockSize := ABlockSize;
  
  FBlocksPerChunk := ABlocksPerChunk;
  FChunkSize := FBlockSize * ABlocksPerChunk;
  FFreeList := nil;
  FAllocCount := 0;
  FFreeCount := 0;
  FLock := TCriticalSection.Create;
  SetLength(FChunks, 0);
end;

destructor TFixedPool.Destroy;
var
  I: Integer;
begin
  FLock.Enter;
  try
    for I := 0 to High(FChunks) do
      System.FreeMem(FChunks[I]);
  finally
    FLock.Leave;
  end;
  FLock.Free;
  inherited;
end;

procedure TFixedPool.AllocateChunk;
var
  ChunkMem: PByte;
  Block: PBlock;
  I: Integer;
begin
  { Allocate Chunk ใหม่ }
  ChunkMem := System.GetMem(FChunkSize);
  SetLength(FChunks, Length(FChunks) + 1);
  FChunks[High(FChunks)] := ChunkMem;
  
  { แบ่ง Chunk เป็น Blocks แล้วใส่ Free List }
  for I := 0 to FBlocksPerChunk - 1 do
  begin
    Block := PBlock(ChunkMem + (I * FBlockSize));
    Block^.Next := FFreeList;
    FFreeList := Block;
  end;
end;

function TFixedPool.Alloc: Pointer;
var
  Block: PBlock;
begin
  FLock.Enter;
  try
    if FFreeList = nil then
      AllocateChunk;
    
    Block := FFreeList;
    FFreeList := Block^.Next;
    Inc(FAllocCount);
    Result := Block;
  finally
    FLock.Leave;
  end;
end;

procedure TFixedPool.Free(APtr: Pointer);
var
  Block: PBlock;
begin
  if APtr = nil then Exit;
  
  FLock.Enter;
  try
    Block := PBlock(APtr);
    Block^.Next := FFreeList;
    FFreeList := Block;
    Inc(FFreeCount);
  finally
    FLock.Leave;
  end;
end;

function TFixedPool.GetLiveCount: Int64;
begin
  Result := FAllocCount - FFreeCount;
end;

procedure TFixedPool.PrintStats;
begin
  WriteLn(Format('Pool(size=%d): alloc=%d, free=%d, live=%d, chunks=%d',
    [FBlockSize, FAllocCount, FFreeCount, GetLiveCount, Length(FChunks)]));
end;

{ TSlabAllocator }
constructor TSlabAllocator.Create;
const
  { Standard Slab Sizes }
  SlabSizes: array[0..9] of Integer = (8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096);
var
  I: Integer;
begin
  FSlabCount := Length(SlabSizes);
  SetLength(FSlabs, FSlabCount);
  
  for I := 0 to FSlabCount - 1 do
  begin
    FSlabs[I].BlockSize := SlabSizes[I];
    FSlabs[I].Pool := TFixedPool.Create(SlabSizes[I]);
  end;
end;

destructor TSlabAllocator.Destroy;
var
  I: Integer;
begin
  for I := 0 to FSlabCount - 1 do
    FSlabs[I].Pool.Free;
  inherited;
end;

function TSlabAllocator.FindSlab(ASize: Integer): Integer;
var I: Integer;
begin
  Result := -1;
  for I := 0 to FSlabCount - 1 do
    if FSlabs[I].BlockSize >= ASize then
    begin
      Result := I;
      Exit;
    end;
end;

function TSlabAllocator.Alloc(ASize: Integer): Pointer;
var
  SlabIdx: Integer;
begin
  SlabIdx := FindSlab(ASize);
  if SlabIdx >= 0 then
    Result := FSlabs[SlabIdx].Pool.Alloc
  else
    Result := System.GetMem(ASize);  { Fallback สำหรับ Size ใหญ่ }
end;

procedure TSlabAllocator.Free(APtr: Pointer; ASize: Integer);
var
  SlabIdx: Integer;
begin
  if APtr = nil then Exit;
  SlabIdx := FindSlab(ASize);
  if SlabIdx >= 0 then
    FSlabs[SlabIdx].Pool.Free(APtr)
  else
    System.FreeMem(APtr);
end;

{ TArenaAllocator }
constructor TArenaAllocator.Create(ACapacity: NativeUInt);
begin
  FCapacity := ACapacity;
  FMemory := System.GetMem(ACapacity);
  FOffset := 0;
  FMarkCount := 0;
  SetLength(FMarks, 64);
end;

destructor TArenaAllocator.Destroy;
begin
  System.FreeMem(FMemory);
  inherited;
end;

function TArenaAllocator.Alloc(ASize: NativeUInt; AAlignment: NativeUInt): Pointer;
var
  AlignedOffset: NativeUInt;
  Mask: NativeUInt;
begin
  { Align ตาม Boundary }
  Mask := AAlignment - 1;
  AlignedOffset := (FOffset + Mask) and not Mask;
  
  if AlignedOffset + ASize > FCapacity then
    raise EOutOfMemory.CreateFmt(
      'Arena Allocator หมด: ต้องการ %d, มีแค่ %d',
      [ASize, FCapacity - AlignedOffset]);
  
  Result := PByte(FMemory) + AlignedOffset;
  FOffset := AlignedOffset + ASize;
end;

procedure TArenaAllocator.Mark;
begin
  if FMarkCount >= Length(FMarks) then
    SetLength(FMarks, Length(FMarks) * 2);
  FMarks[FMarkCount] := FOffset;
  Inc(FMarkCount);
end;

procedure TArenaAllocator.PopMark;
begin
  if FMarkCount > 0 then
  begin
    Dec(FMarkCount);
    FOffset := FMarks[FMarkCount];
  end;
end;

procedure TArenaAllocator.Reset;
begin
  FOffset := 0;
  FMarkCount := 0;
end;

function TArenaAllocator.GetAvailable: NativeUInt;
begin
  Result := FCapacity - FOffset;
end;

end.
```

---

## 5. Weak References และ Circular Reference

```pascal
unit WeakReferences;

{$mode objfpc}{$H+}

interface

uses SysUtils, SyncObjs;

type
  { ตัวอย่าง Circular Reference Problem }
  TParent = class;
  TChild = class;

  { แบบ ผิด: จะไม่ถูก Free เพราะ Circular Reference }
  TParentBad = class(TInterfacedObject)
  public
    Child: IInterface;  { Strong reference ไปยัง Child }
    destructor Destroy; override;
  end;

  TChildBad = class(TInterfacedObject)
  public
    Parent: IInterface;  { Strong reference ไปยัง Parent → CIRCULAR! }
    destructor Destroy; override;
  end;

  { Weak Reference Container }
  generic TWeakRef<T: class> = class
  private
    FObj: T;
    FValid: Boolean;
    
  public
    constructor Create(AObj: T);
    
    function IsAlive: Boolean;
    function Lock: T;  { ได้ Object หรือ nil }
    procedure Invalidate;
    
    property Valid: Boolean read FValid;
  end;

  { แบบถูกต้อง: ใช้ Weak Reference }
  TParent = class
  public
    Child: TChild;  { Strong reference }
    destructor Destroy; override;
  end;

  TChild = class
  public
    ParentRef: specialize TWeakRef<TParent>;  { Weak reference }
    destructor Destroy; override;
  end;

  { Observer Pattern ที่ใช้ Weak References }
  IObserver = interface
    ['{11223344-5566-7788-AABB-CCDDEEFF0011}']
    procedure Update(const AEvent: string);
  end;

  TWeakObserverList = class
  private type
    TWeakObserverEntry = record
      Ref: IObserver;  { ใน Pascal Interface มี RefCount อยู่แล้ว }
    end;
    
  private
    FObservers: array of TWeakObserverEntry;
    FCount: Integer;
    FLock: TCriticalSection;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Add(AObserver: IObserver);
    procedure Remove(AObserver: IObserver);
    procedure Notify(const AEvent: string);
    
    property Count: Integer read FCount;
  end;

implementation

{ TParentBad / TChildBad }
destructor TParentBad.Destroy;
begin
  WriteLn('TParentBad.Destroy');
  inherited;
end;

destructor TChildBad.Destroy;
begin
  WriteLn('TChildBad.Destroy');
  inherited;
end;

{ TWeakRef<T> }
constructor specialize TWeakRef<T>.Create(AObj: T);
begin
  FObj := AObj;
  FValid := AObj <> nil;
end;

function specialize TWeakRef<T>.IsAlive: Boolean;
begin
  Result := FValid and (FObj <> nil);
end;

function specialize TWeakRef<T>.Lock: T;
begin
  if FValid then
    Result := FObj
  else
    Result := nil;
end;

procedure specialize TWeakRef<T>.Invalidate;
begin
  FObj := nil;
  FValid := False;
end;

{ TParent / TChild (ถูกต้อง) }
destructor TParent.Destroy;
begin
  WriteLn('TParent.Destroy - Freeing Child');
  
  { Invalidate ก่อน Free }
  if Child <> nil then
    Child.ParentRef.Invalidate;
  
  Child.Free;
  inherited;
end;

destructor TChild.Destroy;
begin
  WriteLn('TChild.Destroy');
  ParentRef.Free;
  inherited;
end;

{ TWeakObserverList }
constructor TWeakObserverList.Create;
begin
  FLock := TCriticalSection.Create;
  FCount := 0;
end;

destructor TWeakObserverList.Destroy;
begin
  FLock.Free;
  inherited;
end;

procedure TWeakObserverList.Add(AObserver: IObserver);
begin
  FLock.Enter;
  try
    if FCount >= Length(FObservers) then
      SetLength(FObservers, Max(8, Length(FObservers) * 2));
    FObservers[FCount].Ref := AObserver;
    Inc(FCount);
  finally
    FLock.Leave;
  end;
end;

procedure TWeakObserverList.Remove(AObserver: IObserver);
var I: Integer;
begin
  FLock.Enter;
  try
    for I := 0 to FCount - 1 do
      if FObservers[I].Ref = AObserver then
      begin
        FObservers[I] := FObservers[FCount - 1];
        Dec(FCount);
        Break;
      end;
  finally
    FLock.Leave;
  end;
end;

procedure TWeakObserverList.Notify(const AEvent: string);
var
  I: Integer;
  Observer: IObserver;
begin
  FLock.Enter;
  try
    for I := 0 to FCount - 1 do
    begin
      Observer := FObservers[I].Ref;
      if Observer <> nil then
        Observer.Update(AEvent);
    end;
  finally
    FLock.Leave;
  end;
end;

end.
```

---

## 6. Memory Debugging Tools

```pascal
unit MemoryDebug;

{$mode objfpc}{$H+}

interface

uses SysUtils, Classes;

type
  { Memory Usage Reporter }
  TMemoryReporter = class
  public
    class function GetProcessMemory: Int64;
    class function GetVirtualMemory: Int64;
    class procedure PrintMemoryReport;
    class procedure DumpHeapInfo;
  end;

  { Canary-based Buffer Overflow Detection }
  TBufferGuard = class
  private
    FBuffer: PByte;
    FSize: NativeUInt;
    FCanaryValue: Cardinal;
    
    function CheckCanary: Boolean;
    
  public
    constructor Create(ASize: NativeUInt);
    destructor Destroy; override;
    
    function GetBuffer: Pointer;
    procedure Validate;
    
    property Size: NativeUInt read FSize;
  end;

  { Memory Access Pattern Analyzer }
  TAccessPattern = record
    Address: Pointer;
    Size: NativeUInt;
    Timestamp: TDateTime;
    IsWrite: Boolean;
    StackDepth: Integer;
  end;

  TMemoryAnalyzer = class
  private
    FPatterns: array of TAccessPattern;
    FCount: Integer;
    FEnabled: Boolean;
    
  public
    procedure Enable;
    procedure Disable;
    procedure RecordAccess(AAddr: Pointer; ASize: NativeUInt; AIsWrite: Boolean);
    procedure PrintHotSpots(ATopN: Integer = 10);
    procedure Clear;
  end;

  { Object Lifecycle Tracker }
  TObjectTracker = class
  private
    FObjects: TStringList;
    FCreateCount: Int64;
    FDestroyCount: Int64;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure TrackCreate(AObj: TObject; const ATag: string = '');
    procedure TrackDestroy(AObj: TObject);
    procedure PrintReport;
    procedure CheckLeaks;
    
    property LiveCount: Int64 read GetLiveCount;
    function GetLiveCount: Int64;
  end;

implementation

{ TMemoryReporter }
class function TMemoryReporter.GetProcessMemory: Int64;
{$IFDEF LINUX}
var
  F: TextFile;
  Line: string;
  Parts: TStringArray;
begin
  Result := 0;
  AssignFile(F, '/proc/self/status');
  try
    Reset(F);
    while not EOF(F) do
    begin
      ReadLn(F, Line);
      if Pos('VmRSS:', Line) = 1 then
      begin
        Parts := Line.Split([' ', #9], TStringSplitOptions.ExcludeEmpty);
        if Length(Parts) >= 2 then
          Result := StrToInt64Def(Parts[1], 0) * 1024;
        Break;
      end;
    end;
  finally
    CloseFile(F);
  end;
end;
{$ELSE}
begin
  Result := -1;  { Not implemented }
end;
{$ENDIF}

class function TMemoryReporter.GetVirtualMemory: Int64;
{$IFDEF LINUX}
var
  F: TextFile;
  Line: string;
  Parts: TStringArray;
begin
  Result := 0;
  AssignFile(F, '/proc/self/status');
  try
    Reset(F);
    while not EOF(F) do
    begin
      ReadLn(F, Line);
      if Pos('VmSize:', Line) = 1 then
      begin
        Parts := Line.Split([' ', #9], TStringSplitOptions.ExcludeEmpty);
        if Length(Parts) >= 2 then
          Result := StrToInt64Def(Parts[1], 0) * 1024;
        Break;
      end;
    end;
  finally
    CloseFile(F);
  end;
end;
{$ELSE}
begin
  Result := -1;
end;
{$ENDIF}

class procedure TMemoryReporter.PrintMemoryReport;
var
  HeapStatus: TFPCHeapStatus;
  RSS, Virt: Int64;
begin
  HeapStatus := GetFPCHeapStatus;
  RSS := GetProcessMemory;
  Virt := GetVirtualMemory;
  
  WriteLn('=== Memory Report ===');
  WriteLn(Format('Heap Current Use:    %12d bytes (%d KB)',
    [HeapStatus.CurrHeapUsed, HeapStatus.CurrHeapUsed div 1024]));
  WriteLn(Format('Heap Max Use:        %12d bytes (%d KB)',
    [HeapStatus.MaxHeapUsed, HeapStatus.MaxHeapUsed div 1024]));
  WriteLn(Format('Heap Free:           %12d bytes (%d KB)',
    [HeapStatus.CurrHeapFree, HeapStatus.CurrHeapFree div 1024]));
  
  if RSS > 0 then
    WriteLn(Format('Process RSS:         %12d bytes (%d MB)',
      [RSS, RSS div (1024*1024)]));
  
  if Virt > 0 then
    WriteLn(Format('Virtual Memory:      %12d bytes (%d MB)',
      [Virt, Virt div (1024*1024)]));
end;

class procedure TMemoryReporter.DumpHeapInfo;
begin
  { ใช้ heaptrc ถ้าเปิดใช้งาน }
  DumpHeap;
end;

{ TBufferGuard }
constructor TBufferGuard.Create(ASize: NativeUInt);
const
  CANARY_BEFORE = $DEADBEEF;
  CANARY_AFTER  = $CAFEBABE;
begin
  FSize := ASize;
  FCanaryValue := CANARY_BEFORE;
  
  { Allocate: Canary Before + Buffer + Canary After }
  FBuffer := System.GetMem(ASize + SizeOf(Cardinal) * 2);
  
  { วาง Canary ก่อน Buffer }
  PCardinal(FBuffer)^ := CANARY_BEFORE;
  
  { วาง Canary หลัง Buffer }
  PCardinal(FBuffer + SizeOf(Cardinal) + ASize)^ := CANARY_AFTER;
  
  { Clear Buffer }
  FillChar((FBuffer + SizeOf(Cardinal))^, ASize, 0);
end;

destructor TBufferGuard.Destroy;
begin
  Validate;  { ตรวจสอบก่อน Free }
  System.FreeMem(FBuffer);
  inherited;
end;

function TBufferGuard.CheckCanary: Boolean;
const
  CANARY_BEFORE = $DEADBEEF;
  CANARY_AFTER  = $CAFEBABE;
begin
  Result := (PCardinal(FBuffer)^ = CANARY_BEFORE) and
            (PCardinal(FBuffer + SizeOf(Cardinal) + FSize)^ = CANARY_AFTER);
end;

function TBufferGuard.GetBuffer: Pointer;
begin
  Result := FBuffer + SizeOf(Cardinal);
end;

procedure TBufferGuard.Validate;
begin
  if not CheckCanary then
    raise EAccessViolation.Create(
      'Buffer Overflow ตรวจพบ! Canary ถูกเขียนทับ');
end;

{ TObjectTracker }
constructor TObjectTracker.Create;
begin
  FObjects := TStringList.Create;
  FObjects.Sorted := True;
  FCreateCount := 0;
  FDestroyCount := 0;
end;

destructor TObjectTracker.Destroy;
begin
  FObjects.Free;
  inherited;
end;

procedure TObjectTracker.TrackCreate(AObj: TObject; const ATag: string);
var
  Key: string;
begin
  Key := Format('%p|%s|%s', [AObj, AObj.ClassName, ATag]);
  FObjects.Add(Key);
  Inc(FCreateCount);
end;

procedure TObjectTracker.TrackDestroy(AObj: TObject);
var
  I: Integer;
  Prefix: string;
begin
  Prefix := Format('%p|', [AObj]);
  for I := FObjects.Count - 1 downto 0 do
    if Pos(Prefix, FObjects[I]) = 1 then
    begin
      FObjects.Delete(I);
      Inc(FDestroyCount);
      Exit;
    end;
end;

procedure TObjectTracker.PrintReport;
begin
  WriteLn('=== Object Tracker Report ===');
  WriteLn('Created:  ', FCreateCount);
  WriteLn('Destroyed:', FDestroyCount);
  WriteLn('Live:     ', FCreateCount - FDestroyCount);
  
  if FObjects.Count > 0 then
  begin
    WriteLn('--- Live Objects ---');
    WriteLn(FObjects.Text);
  end;
end;

procedure TObjectTracker.CheckLeaks;
begin
  if FObjects.Count > 0 then
  begin
    WriteLn('*** WARNING: ', FObjects.Count, ' Object(s) ยังไม่ถูก Free! ***');
    PrintReport;
  end
  else
    WriteLn('No memory leaks detected.');
end;

function TObjectTracker.GetLiveCount: Int64;
begin
  Result := FCreateCount - FDestroyCount;
end;

{ TMemoryAnalyzer }
procedure TMemoryAnalyzer.Enable;
begin
  FEnabled := True;
end;

procedure TMemoryAnalyzer.Disable;
begin
  FEnabled := False;
end;

procedure TMemoryAnalyzer.RecordAccess(AAddr: Pointer; ASize: NativeUInt;
  AIsWrite: Boolean);
begin
  if not FEnabled then Exit;
  
  if FCount >= Length(FPatterns) then
    SetLength(FPatterns, Max(1000, Length(FPatterns) * 2));
  
  FPatterns[FCount].Address := AAddr;
  FPatterns[FCount].Size := ASize;
  FPatterns[FCount].Timestamp := Now;
  FPatterns[FCount].IsWrite := AIsWrite;
  Inc(FCount);
end;

procedure TMemoryAnalyzer.PrintHotSpots(ATopN: Integer);
{ วิเคราะห์ Access Pattern }
begin
  WriteLn(Format('Total Memory Accesses: %d', [FCount]));
  { Implementation ขึ้นอยู่กับ Analysis Method }
end;

procedure TMemoryAnalyzer.Clear;
begin
  FCount := 0;
  SetLength(FPatterns, 0);
end;

end.
```

---

## 7. โปรแกรมตัวอย่างครบถ้วน: Memory Pool Implementation

```pascal
program MemoryPoolDemo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, SyncObjs, DateUtils;

{ ===== Typed Memory Pool ===== }
type
  generic TTypedPool<T> = class
  private type
    PItem = ^T;
    PFreeNode = ^TFreeNode;
    TFreeNode = record
      Next: PFreeNode;
    end;
    
  private
    FPool: array of T;
    FFreeList: PFreeNode;
    FCapacity: Integer;
    FUsed: Integer;
    FLock: TCriticalSection;
    
    procedure Grow;
    
  public
    constructor Create(AInitialCapacity: Integer = 64);
    destructor Destroy; override;
    
    function Acquire: PItem;
    procedure Release(AItem: PItem);
    
    property Used: Integer read FUsed;
    property Capacity: Integer read FCapacity;
  end;

{ ===== Application: Particle System ===== }
type
  TParticle = record
    X, Y: Double;
    VX, VY: Double;       { Velocity }
    Life: Double;         { 0.0 - 1.0 }
    Size: Double;
    Red, Green, Blue: Byte;
    Active: Boolean;
  end;

  PParticle = ^TParticle;

  TParticleSystem = class
  private
    FPool: specialize TTypedPool<TParticle>;
    FActive: array of PParticle;
    FActiveCount: Integer;
    FTime: Double;
    
    procedure UpdateParticle(AParticle: PParticle; ADelta: Double);
    
  public
    constructor Create(AMaxParticles: Integer);
    destructor Destroy; override;
    
    procedure Emit(AX, AY: Double; ACount: Integer);
    procedure Update(ADelta: Double);
    procedure PrintStats;
    
    property ActiveCount: Integer read FActiveCount;
  end;

  { Memory Usage Benchmark }
  TMemoryBenchmark = class
  public
    class procedure Run;
    class procedure BenchmarkNew(ACount: Integer);
    class procedure BenchmarkPool(ACount: Integer);
  end;

{ TTypedPool<T> }
constructor specialize TTypedPool<T>.Create(AInitialCapacity: Integer);
begin
  FCapacity := AInitialCapacity;
  SetLength(FPool, FCapacity);
  FFreeList := nil;
  FUsed := 0;
  FLock := TCriticalSection.Create;
  Grow;  { Pre-allocate }
end;

destructor specialize TTypedPool<T>.Destroy;
begin
  FLock.Free;
  inherited;
end;

procedure specialize TTypedPool<T>.Grow;
var
  OldCap, I: Integer;
  Node: PFreeNode;
begin
  OldCap := FCapacity;
  FCapacity := FCapacity * 2;
  SetLength(FPool, FCapacity);
  
  { เพิ่ม Items ใหม่เข้า Free List }
  for I := OldCap to FCapacity - 1 do
  begin
    Node := PFreeNode(@FPool[I]);
    Node^.Next := FFreeList;
    FFreeList := Node;
  end;
end;

function specialize TTypedPool<T>.Acquire: PItem;
var Node: PFreeNode;
begin
  FLock.Enter;
  try
    if FFreeList = nil then
      Grow;
    
    Node := FFreeList;
    FFreeList := Node^.Next;
    Inc(FUsed);
    
    Result := PItem(Node);
    FillChar(Result^, SizeOf(T), 0);  { Initialize }
  finally
    FLock.Leave;
  end;
end;

procedure specialize TTypedPool<T>.Release(AItem: PItem);
var Node: PFreeNode;
begin
  if AItem = nil then Exit;
  
  FLock.Enter;
  try
    Node := PFreeNode(AItem);
    Node^.Next := FFreeList;
    FFreeList := Node;
    Dec(FUsed);
  finally
    FLock.Leave;
  end;
end;

{ TParticleSystem }
constructor TParticleSystem.Create(AMaxParticles: Integer);
begin
  FPool := specialize TTypedPool<TParticle>.Create(AMaxParticles);
  SetLength(FActive, AMaxParticles);
  FActiveCount := 0;
  FTime := 0;
end;

destructor TParticleSystem.Destroy;
var I: Integer;
begin
  { Return all active particles to pool }
  for I := 0 to FActiveCount - 1 do
    FPool.Release(FActive[I]);
  FPool.Free;
  inherited;
end;

procedure TParticleSystem.Emit(AX, AY: Double; ACount: Integer);
var
  I: Integer;
  P: PParticle;
  Angle, Speed: Double;
begin
  for I := 1 to ACount do
  begin
    if FActiveCount >= Length(FActive) then Break;
    
    P := FPool.Acquire;
    P^.X := AX;
    P^.Y := AY;
    
    { Random velocity }
    Angle := Random * 2 * Pi;
    Speed := 10 + Random * 40;
    P^.VX := Cos(Angle) * Speed;
    P^.VY := Sin(Angle) * Speed;
    
    P^.Life := 1.0;
    P^.Size := 2 + Random * 8;
    P^.Red := Random(256);
    P^.Green := Random(256);
    P^.Blue := Random(256);
    P^.Active := True;
    
    FActive[FActiveCount] := P;
    Inc(FActiveCount);
  end;
end;

procedure TParticleSystem.UpdateParticle(AParticle: PParticle; ADelta: Double);
begin
  AParticle^.X := AParticle^.X + AParticle^.VX * ADelta;
  AParticle^.Y := AParticle^.Y + AParticle^.VY * ADelta;
  AParticle^.VY := AParticle^.VY + 98 * ADelta;  { Gravity }
  AParticle^.Life := AParticle^.Life - ADelta * 0.5;
  AParticle^.Size := AParticle^.Size * (1 - ADelta * 0.2);
  
  if AParticle^.Life <= 0 then
    AParticle^.Active := False;
end;

procedure TParticleSystem.Update(ADelta: Double);
var
  I, J: Integer;
begin
  FTime := FTime + ADelta;
  J := 0;
  
  for I := 0 to FActiveCount - 1 do
  begin
    UpdateParticle(FActive[I], ADelta);
    
    if FActive[I]^.Active then
    begin
      FActive[J] := FActive[I];
      Inc(J);
    end
    else
    begin
      FPool.Release(FActive[I]);  { Return to pool }
    end;
  end;
  
  FActiveCount := J;
end;

procedure TParticleSystem.PrintStats;
begin
  WriteLn(Format('Particles: Active=%d, Pool Used=%d/%d',
    [FActiveCount, FPool.Used, FPool.Capacity]));
end;

{ TMemoryBenchmark }
class procedure TMemoryBenchmark.Run;
const
  COUNT = 10000;
  ITERATIONS = 5;
var
  I: Integer;
  TimeNew, TimePool: Int64;
  
  function MeasureMs(AProc: TProcedure): Int64;
  var Start: TDateTime;
  begin
    Start := Now;
    AProc;
    Result := MilliSecondsBetween(Now, Start);
  end;

begin
  WriteLn('=== Memory Benchmark ===');
  WriteLn('Count = ', COUNT, ', Iterations = ', ITERATIONS);
  WriteLn;
  
  TimeNew := 0;
  TimePool := 0;
  
  for I := 1 to ITERATIONS do
  begin
    TimeNew := TimeNew + MeasureMs(procedure begin BenchmarkNew(COUNT); end);
    TimePool := TimePool + MeasureMs(procedure begin BenchmarkPool(COUNT); end);
  end;
  
  WriteLn(Format('New/Free avg:  %d ms', [TimeNew div ITERATIONS]));
  WriteLn(Format('Pool avg:      %d ms', [TimePool div ITERATIONS]));
  WriteLn(Format('Speedup:       %.1fx', [TimeNew / Max(1, TimePool)]));
end;

class procedure TMemoryBenchmark.BenchmarkNew(ACount: Integer);
var
  Particles: array of PParticle;
  I: Integer;
begin
  SetLength(Particles, ACount);
  for I := 0 to ACount - 1 do
  begin
    New(Particles[I]);
    Particles[I]^.X := Random * 1000;
    Particles[I]^.Y := Random * 1000;
    Particles[I]^.Life := 1.0;
  end;
  for I := 0 to ACount - 1 do
    Dispose(Particles[I]);
end;

class procedure TMemoryBenchmark.BenchmarkPool(ACount: Integer);
var
  Pool: specialize TTypedPool<TParticle>;
  Particles: array of PParticle;
  I: Integer;
begin
  Pool := specialize TTypedPool<TParticle>.Create(ACount);
  SetLength(Particles, ACount);
  try
    for I := 0 to ACount - 1 do
    begin
      Particles[I] := Pool.Acquire;
      Particles[I]^.X := Random * 1000;
      Particles[I]^.Y := Random * 1000;
      Particles[I]^.Life := 1.0;
    end;
    for I := 0 to ACount - 1 do
      Pool.Release(Particles[I]);
  finally
    Pool.Free;
  end;
end;

{ Main Demo }
var
  PS: TParticleSystem;
  Step: Integer;
begin
  Randomize;
  
  { Demo 1: Particle System }
  WriteLn('=== Particle System Demo ===');
  PS := TParticleSystem.Create(10000);
  try
    PS.Emit(400, 300, 500);
    PS.PrintStats;
    
    for Step := 1 to 10 do
    begin
      PS.Update(0.016);  { 60 FPS }
      PS.Emit(400, 300, 50);  { ปล่อย Particle ใหม่ }
    end;
    
    PS.PrintStats;
  finally
    PS.Free;
  end;
  
  WriteLn;
  
  { Demo 2: Benchmark }
  TMemoryBenchmark.Run;
end.
```

---

## 8. Large Memory Application Patterns

```pascal
unit LargeMemoryPatterns;

{$mode objfpc}{$H+}

interface

uses SysUtils, Classes;

type
  { Memory-Mapped Virtual Array }
  generic TVirtualArray<T> = class
  private type
    TPage = class
      Data: array of T;
      LastAccess: TDateTime;
      Dirty: Boolean;
      constructor Create(ASize: Integer);
    end;
    
  private
    FPages: array of TPage;
    FPageCount: Integer;
    FItemsPerPage: Integer;
    FMaxLoadedPages: Integer;
    FLoadedCount: Integer;
    FFileName: string;
    
    function GetPage(APageIdx: Integer): TPage;
    procedure EvictPage;
    function ItemsInLastPage: Integer;
    
  public
    constructor Create(AFileName: string; ATotalItems: Int64;
                        AItemsPerPage: Integer = 4096;
                        AMaxLoadedPages: Integer = 32);
    destructor Destroy; override;
    
    function GetItem(AIndex: Int64): T;
    procedure SetItem(AIndex: Int64; const AValue: T);
    procedure Flush;
    
    property Count: Int64 read GetCount;
    function GetCount: Int64;
  end;

  { Chunked Buffer สำหรับ Streaming }
  TChunkedBuffer = class
  private const
    DEFAULT_CHUNK_SIZE = 64 * 1024;  { 64 KB }
    
  private type
    TChunk = class
      Data: array of Byte;
      Used: Integer;
      constructor Create(ASize: Integer);
    end;
    
  private
    FChunks: TObjectList;
    FChunkSize: Integer;
    FTotalSize: Int64;
    FReadPos: Int64;
    
  public
    constructor Create(AChunkSize: Integer = DEFAULT_CHUNK_SIZE);
    destructor Destroy; override;
    
    procedure Write(const AData; ASize: NativeUInt);
    function Read(var AData; ASize: NativeUInt): NativeUInt;
    procedure Seek(APos: Int64);
    procedure Clear;
    
    property Size: Int64 read FTotalSize;
    property Position: Int64 read FReadPos;
  end;

  { Copy-on-Write String Buffer }
  TCOWString = class
  private type
    TStringData = class
      Data: string;
      RefCount: Integer;
      constructor Create(const S: string);
    end;
    
  private
    FData: TStringData;
    
    procedure Detach;
    
  public
    constructor Create(const S: string = '');
    constructor CreateShared(ASource: TCOWString);
    destructor Destroy; override;
    
    procedure Append(const S: string);
    function ToString: string; override;
    function Length: Integer;
    
    property Value: string read GetValue;
    function GetValue: string;
  end;

implementation

{ TPage }
constructor specialize TVirtualArray<T>.TPage.Create(ASize: Integer);
begin
  SetLength(Data, ASize);
  LastAccess := Now;
  Dirty := False;
end;

{ TVirtualArray<T> }
constructor specialize TVirtualArray<T>.Create(AFileName: string; ATotalItems: Int64;
  AItemsPerPage, AMaxLoadedPages: Integer);
begin
  FFileName := AFileName;
  FItemsPerPage := AItemsPerPage;
  FMaxLoadedPages := AMaxLoadedPages;
  FLoadedCount := 0;
  
  FPageCount := (ATotalItems + AItemsPerPage - 1) div AItemsPerPage;
  SetLength(FPages, FPageCount);
  
  { ไม่ Load ทันที - Lazy Loading }
end;

destructor specialize TVirtualArray<T>.Destroy;
begin
  Flush;
  { Free loaded pages }
  inherited;
end;

function specialize TVirtualArray<T>.GetCount: Int64;
begin
  Result := Int64(FPageCount) * FItemsPerPage;
end;

function specialize TVirtualArray<T>.GetPage(APageIdx: Integer): TPage;
begin
  if FPages[APageIdx] = nil then
  begin
    if FLoadedCount >= FMaxLoadedPages then
      EvictPage;
    
    FPages[APageIdx] := TPage.Create(FItemsPerPage);
    Inc(FLoadedCount);
    
    { Load จาก Disk ถ้ามี }
    { ... }
  end;
  
  FPages[APageIdx].LastAccess := Now;
  Result := FPages[APageIdx];
end;

procedure specialize TVirtualArray<T>.EvictPage;
var
  I, OldestIdx: Integer;
  OldestTime: TDateTime;
begin
  OldestIdx := -1;
  OldestTime := Now;
  
  for I := 0 to FPageCount - 1 do
    if (FPages[I] <> nil) and (FPages[I].LastAccess < OldestTime) then
    begin
      OldestTime := FPages[I].LastAccess;
      OldestIdx := I;
    end;
  
  if OldestIdx >= 0 then
  begin
    if FPages[OldestIdx].Dirty then
    begin
      { Save to Disk }
      { ... }
    end;
    FPages[OldestIdx].Free;
    FPages[OldestIdx] := nil;
    Dec(FLoadedCount);
  end;
end;

function specialize TVirtualArray<T>.GetItem(AIndex: Int64): T;
var
  PageIdx: Integer;
  ItemIdx: Integer;
begin
  PageIdx := AIndex div FItemsPerPage;
  ItemIdx := AIndex mod FItemsPerPage;
  Result := GetPage(PageIdx).Data[ItemIdx];
end;

procedure specialize TVirtualArray<T>.SetItem(AIndex: Int64; const AValue: T);
var
  PageIdx: Integer;
  ItemIdx: Integer;
  Page: TPage;
begin
  PageIdx := AIndex div FItemsPerPage;
  ItemIdx := AIndex mod FItemsPerPage;
  Page := GetPage(PageIdx);
  Page.Data[ItemIdx] := AValue;
  Page.Dirty := True;
end;

procedure specialize TVirtualArray<T>.Flush;
var I: Integer;
begin
  for I := 0 to FPageCount - 1 do
    if (FPages[I] <> nil) and FPages[I].Dirty then
    begin
      { Save to Disk }
      FPages[I].Dirty := False;
    end;
end;

function specialize TVirtualArray<T>.ItemsInLastPage: Integer;
begin
  Result := FItemsPerPage;  { Simple for now }
end;

{ TChunkedBuffer.TChunk }
constructor TChunkedBuffer.TChunk.Create(ASize: Integer);
begin
  SetLength(Data, ASize);
  Used := 0;
end;

{ TChunkedBuffer }
constructor TChunkedBuffer.Create(AChunkSize: Integer);
begin
  FChunkSize := AChunkSize;
  FChunks := TObjectList.Create(True);
  FTotalSize := 0;
  FReadPos := 0;
end;

destructor TChunkedBuffer.Destroy;
begin
  FChunks.Free;
  inherited;
end;

procedure TChunkedBuffer.Write(const AData; ASize: NativeUInt);
var
  Src: PByte;
  Remaining, ToWrite: NativeUInt;
  Chunk: TChunk;
begin
  Src := @AData;
  Remaining := ASize;
  
  while Remaining > 0 do
  begin
    { Get or create current chunk }
    if (FChunks.Count = 0) or
       (TChunk(FChunks.Last).Used >= FChunkSize) then
    begin
      Chunk := TChunk.Create(FChunkSize);
      FChunks.Add(Chunk);
    end
    else
      Chunk := TChunk(FChunks.Last);
    
    ToWrite := Min(Remaining, NativeUInt(FChunkSize - Chunk.Used));
    Move(Src^, Chunk.Data[Chunk.Used], ToWrite);
    Inc(Chunk.Used, ToWrite);
    Inc(FTotalSize, ToWrite);
    Inc(Src, ToWrite);
    Dec(Remaining, ToWrite);
  end;
end;

function TChunkedBuffer.Read(var AData; ASize: NativeUInt): NativeUInt;
var
  Dst: PByte;
  Remaining, ToRead: NativeUInt;
  ChunkIdx: Integer;
  OffsetInChunk: Int64;
  Chunk: TChunk;
begin
  Result := 0;
  Dst := @AData;
  Remaining := Min(ASize, NativeUInt(FTotalSize - FReadPos));
  
  while Remaining > 0 do
  begin
    ChunkIdx := FReadPos div FChunkSize;
    OffsetInChunk := FReadPos mod FChunkSize;
    
    if ChunkIdx >= FChunks.Count then Break;
    
    Chunk := TChunk(FChunks[ChunkIdx]);
    ToRead := Min(Remaining, NativeUInt(Chunk.Used - OffsetInChunk));
    
    Move(Chunk.Data[OffsetInChunk], Dst^, ToRead);
    Inc(Dst, ToRead);
    Inc(FReadPos, ToRead);
    Inc(Result, ToRead);
    Dec(Remaining, ToRead);
  end;
end;

procedure TChunkedBuffer.Seek(APos: Int64);
begin
  FReadPos := Max(0, Min(APos, FTotalSize));
end;

procedure TChunkedBuffer.Clear;
begin
  FChunks.Clear;
  FTotalSize := 0;
  FReadPos := 0;
end;

{ TCOWString.TStringData }
constructor TCOWString.TStringData.Create(const S: string);
begin
  Data := S;
  RefCount := 1;
end;

{ TCOWString }
constructor TCOWString.Create(const S: string);
begin
  FData := TStringData.Create(S);
end;

constructor TCOWString.CreateShared(ASource: TCOWString);
begin
  FData := ASource.FData;
  Inc(FData.RefCount);
end;

destructor TCOWString.Destroy;
begin
  Dec(FData.RefCount);
  if FData.RefCount = 0 then
    FData.Free;
  inherited;
end;

procedure TCOWString.Detach;
var NewData: TStringData;
begin
  if FData.RefCount > 1 then
  begin
    NewData := TStringData.Create(FData.Data);
    Dec(FData.RefCount);
    FData := NewData;
  end;
end;

procedure TCOWString.Append(const S: string);
begin
  Detach;  { Copy ก่อนแก้ไข }
  FData.Data := FData.Data + S;
end;

function TCOWString.ToString: string;
begin
  Result := FData.Data;
end;

function TCOWString.GetValue: string;
begin
  Result := FData.Data;
end;

function TCOWString.Length: Integer;
begin
  Result := System.Length(FData.Data);
end;

end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Memory Leak Investigation
เขียนโปรแกรมที่มี Memory Leak 3 ประเภท แล้วแก้ไข:
- Object ที่ไม่ถูก Free ใน Exception Path
- Circular Reference ระหว่าง Objects
- Dynamic Array ที่ไม่ถูกล้าง

### ข้อ 2 - Custom Allocator
สร้าง Custom Allocator สำหรับ Game Engine:
- Fixed-Size Pool สำหรับ Entity
- Frame Allocator สำหรับ Temporary Data
- String Interning สำหรับ Asset Names

### ข้อ 3 - Smart Pointer Library
สร้าง Smart Pointer Library สมบูรณ์:
- TUniquePtr (Single Owner)
- TSharedPtr (Reference Counted)
- TWeakPtr (Non-owning)
- Thread-safe Operations

### ข้อ 4 - Memory-Efficient Data Structure
สร้าง Compact Hash Map:
- Open Addressing
- Robin Hood Hashing
- Benchmarkเทียบกับ TDictionary
- Memory Usage Comparison

### ข้อ 5 - Large File Processing
สร้าง Memory-Efficient CSV Parser:
- Stream Processing (ไม่โหลดทั้งไฟล์)
- Memory Pool สำหรับ String Fields
- Progress Reporting
- รองรับไฟล์ขนาด GB
