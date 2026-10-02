# ตอนที่ 86: Scalability กับ Pascal/Lazarus

## บทนำ: การออกแบบระบบที่ Scale ได้

Scalability คือความสามารถของระบบในการรองรับ Load ที่เพิ่มขึ้นโดยยังคงประสิทธิภาพที่ดี

## 1. Horizontal vs Vertical Scaling

```
Vertical Scaling (Scale Up):
┌─────────────────────┐
│  Server             │
│  CPU: 32 cores      │  ← เพิ่ม Hardware
│  RAM: 256 GB        │
│  Storage: 10 TB     │
└─────────────────────┘
ข้อเสีย: มี Limit, ราคาแพง, Single Point of Failure

Horizontal Scaling (Scale Out):
┌─────────┐ ┌─────────┐ ┌─────────┐
│Server 1 │ │Server 2 │ │Server 3 │  ← เพิ่ม Server
│ 8 cores │ │ 8 cores │ │ 8 cores │
└────┬────┘ └────┬────┘ └────┬────┘
     └───────────┼───────────┘
           ┌─────▼─────┐
           │   Load    │
           │ Balancer  │
           └───────────┘
ข้อดี: Scale ได้ไม่จำกัด, High Availability
```

## 2. Connection Pooling

```pascal
// uConnectionPool.pas - Database Connection Pool
unit uConnectionPool;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections, sqldb;

type
  TPooledConnection = class
  private
    FConnection: TSQLConnection;
    FInUse: Boolean;
    FLastUsed: TDateTime;
    FCreateTime: TDateTime;
    FUseCount: Integer;
    FPoolId: Integer;
  public
    constructor Create(APoolId: Integer; const AConnectionString: string);
    destructor Destroy; override;

    function IsAlive: Boolean;
    procedure Reset;

    property Connection: TSQLConnection read FConnection;
    property InUse: Boolean read FInUse write FInUse;
    property LastUsed: TDateTime read FLastUsed write FLastUsed;
    property PoolId: Integer read FPoolId;
    property UseCount: Integer read FUseCount;
  end;

  TConnectionPoolStats = record
    TotalConnections: Integer;
    ActiveConnections: Integer;
    IdleConnections: Integer;
    WaitingRequests: Integer;
    TotalAcquired: Int64;
    TotalReleased: Int64;
    AverageWaitTimeMs: Double;
  end;

  TConnectionPool = class
  private
    FConnections: TObjectList<TPooledConnection>;
    FConnectionString: string;
    FMinConnections: Integer;
    FMaxConnections: Integer;
    FConnectionTimeout: Integer; // milliseconds
    FIdleTimeout: Integer; // seconds
    FMaxLifetime: Integer; // seconds
    FCriticalSection: TCriticalSection;
    FWaitEvent: TEvent;
    FStats: TConnectionPoolStats;
    FMaintenanceThread: TThread;
    FIsShuttingDown: Boolean;
    FNextPoolId: Integer;

    function CreateConnection: TPooledConnection;
    function FindIdleConnection: TPooledConnection;
    procedure RemoveDeadConnections;
    procedure EnsureMinConnections;
    procedure StartMaintenanceThread;

  public
    constructor Create(
      const AConnectionString: string;
      AMinConnections: Integer = 2;
      AMaxConnections: Integer = 20;
      AConnectionTimeout: Integer = 30000;
      AIdleTimeout: Integer = 600;
      AMaxLifetime: Integer = 3600);
    destructor Destroy; override;

    function Acquire: TPooledConnection;
    procedure Release(const AConnection: TPooledConnection);
    function GetStats: TConnectionPoolStats;

    property ConnectionString: string read FConnectionString;
    property MinConnections: Integer read FMinConnections;
    property MaxConnections: Integer read FMaxConnections;
  end;

  // Connection Pool Wrapper - RAII Pattern
  TPoolConnection = class
  private
    FPool: TConnectionPool;
    FConnection: TPooledConnection;
  public
    constructor Create(const APool: TConnectionPool);
    destructor Destroy; override;
    property Connection: TPooledConnection read FConnection;
  end;

implementation

constructor TPooledConnection.Create(APoolId: Integer;
  const AConnectionString: string);
begin
  inherited Create;
  FPoolId := APoolId;
  FInUse := False;
  FCreateTime := Now;
  FLastUsed := Now;
  FUseCount := 0;

  // Create and open connection
  FConnection := TPostgresConnection.Create(nil);
  FConnection.DatabaseName := AConnectionString;
  FConnection.Open;
end;

function TPooledConnection.IsAlive: Boolean;
begin
  try
    Result := FConnection.Connected;
    if Result then
    begin
      // Quick ping test
      var Q := TSQLQuery.Create(nil);
      try
        Q.DataBase := FConnection;
        Q.SQL.Text := 'SELECT 1';
        Q.Open;
        Q.Close;
      finally
        Q.Free;
      end;
    end;
  except
    Result := False;
  end;
end;

constructor TConnectionPool.Create(
  const AConnectionString: string;
  AMinConnections, AMaxConnections,
  AConnectionTimeout, AIdleTimeout, AMaxLifetime: Integer);
begin
  inherited Create;
  FConnectionString := AConnectionString;
  FMinConnections := AMinConnections;
  FMaxConnections := AMaxConnections;
  FConnectionTimeout := AConnectionTimeout;
  FIdleTimeout := AIdleTimeout;
  FMaxLifetime := AMaxLifetime;
  FIsShuttingDown := False;
  FNextPoolId := 1;

  FCriticalSection := TCriticalSection.Create;
  FWaitEvent := TEvent.Create(nil, False, False, '');
  FConnections := TObjectList<TPooledConnection>.Create(True);

  // Create minimum connections
  EnsureMinConnections;
  StartMaintenanceThread;
end;

destructor TConnectionPool.Destroy;
begin
  FIsShuttingDown := True;
  if Assigned(FMaintenanceThread) then
  begin
    FMaintenanceThread.Terminate;
    FMaintenanceThread.WaitFor;
    FMaintenanceThread.Free;
  end;
  FConnections.Free;
  FWaitEvent.Free;
  FCriticalSection.Free;
  inherited Destroy;
end;

function TConnectionPool.Acquire: TPooledConnection;
var
  StartTime: TDateTime;
  ElapsedMs: Int64;
begin
  StartTime := Now;
  Inc(FStats.WaitingRequests);

  repeat
    FCriticalSection.Acquire;
    try
      // Try to find idle connection
      Result := FindIdleConnection;

      if Assigned(Result) then
      begin
        Result.InUse := True;
        Result.LastUsed := Now;
        Inc(Result.FUseCount);
        Inc(FStats.TotalAcquired);
        Inc(FStats.ActiveConnections);
        Dec(FStats.WaitingRequests);

        ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
        FStats.AverageWaitTimeMs := (FStats.AverageWaitTimeMs + ElapsedMs) / 2;
        Exit;
      end;

      // Create new connection if under max
      if FConnections.Count < FMaxConnections then
      begin
        Result := CreateConnection;
        Result.InUse := True;
        Result.LastUsed := Now;
        FConnections.Add(Result);
        Inc(FStats.TotalAcquired);
        Inc(FStats.ActiveConnections);
        Dec(FStats.WaitingRequests);
        Exit;
      end;
    finally
      FCriticalSection.Release;
    end;

    // Wait for a connection to become available
    ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
    if ElapsedMs >= FConnectionTimeout then
    begin
      Dec(FStats.WaitingRequests);
      raise EConnectionPoolTimeout.CreateFmt(
        'Connection pool timeout after %dms', [FConnectionTimeout]);
    end;

    FWaitEvent.WaitFor(100); // Wait 100ms and try again
  until FIsShuttingDown;

  Dec(FStats.WaitingRequests);
  raise EConnectionPoolException.Create('Connection pool is shutting down');
end;

procedure TConnectionPool.Release(const AConnection: TPooledConnection);
begin
  FCriticalSection.Acquire;
  try
    AConnection.InUse := False;
    AConnection.LastUsed := Now;
    Dec(FStats.ActiveConnections);
    Inc(FStats.TotalReleased);
    FWaitEvent.SetEvent; // Signal waiting threads
  finally
    FCriticalSection.Release;
  end;
end;

function TConnectionPool.FindIdleConnection: TPooledConnection;
var
  Conn: TPooledConnection;
begin
  Result := nil;
  for Conn in FConnections do
  begin
    if not Conn.InUse then
    begin
      if Conn.IsAlive then
      begin
        Result := Conn;
        Exit;
      end;
    end;
  end;
end;

function TConnectionPool.CreateConnection: TPooledConnection;
begin
  Result := TPooledConnection.Create(FNextPoolId, FConnectionString);
  Inc(FNextPoolId);
  Inc(FStats.TotalConnections);
  Inc(FStats.IdleConnections);
end;

// RAII Connection Wrapper
constructor TPoolConnection.Create(const APool: TConnectionPool);
begin
  inherited Create;
  FPool := APool;
  FConnection := FPool.Acquire;
end;

destructor TPoolConnection.Destroy;
begin
  if Assigned(FPool) and Assigned(FConnection) then
    FPool.Release(FConnection);
  inherited Destroy;
end;

end.
```

## 3. Redis Cache Integration

```pascal
// uRedisCache.pas - Redis Cache Client
unit uRedisCache;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, fpjson, SyncObjs;

type
  TCacheOptions = record
    Expiry: Integer;    // seconds (0 = no expiry)
    Tags: TArray<string>;
    SlidingExpiry: Boolean;
  end;

  ICache = interface
    function Get(const AKey: string): string;
    function GetObject<T: class>(const AKey: string): T;
    function TryGet<T>(const AKey: string; out AValue: T): Boolean;
    procedure Set(const AKey, AValue: string;
      AExpirySeconds: Integer = 0);
    procedure SetObject<T: class>(const AKey: string; const AValue: T;
      AExpirySeconds: Integer = 0);
    procedure Delete(const AKey: string);
    procedure DeleteByPattern(const APattern: string);
    procedure DeleteByTag(const ATag: string);
    function Exists(const AKey: string): Boolean;
    function GetOrCreate<T>(const AKey: string;
      const AFactory: TFunc<T>;
      AExpirySeconds: Integer = 0): T;
    procedure Invalidate(const ATags: TArray<string>);
    procedure Flush;
  end;

  TRedisClient = class
  private
    FHost: string;
    FPort: Integer;
    FPassword: string;
    FDatabase: Integer;
    FSocket: TSocket;
    FConnected: Boolean;
    FCriticalSection: TCriticalSection;
    FConnectionPool: TList<TSocket>;

    procedure Connect;
    procedure Disconnect;
    function SendCommand(const AArgs: TArray<string>): string;
    function ParseResponse(const AResponse: string): string;
    procedure EnsureConnected;

  public
    constructor Create(const AHost: string = 'localhost';
      APort: Integer = 6379;
      const APassword: string = '';
      ADatabase: Integer = 0);
    destructor Destroy; override;

    // Basic Commands
    function Get(const AKey: string): string;
    procedure SetEx(const AKey, AValue: string; AExpiry: Integer);
    procedure Set_(const AKey, AValue: string);
    procedure Del(const AKey: string);
    function Exists(const AKey: string): Boolean;
    function Keys(const APattern: string): TArray<string>;
    function TTL(const AKey: string): Integer;
    procedure Expire(const AKey: string; AExpiry: Integer);

    // Hash Commands
    function HGet(const AKey, AField: string): string;
    procedure HSet(const AKey, AField, AValue: string);
    procedure HMSet(const AKey: string; const AFields: TStringList);
    function HGetAll(const AKey: string): TStringList;
    function HExists(const AKey, AField: string): Boolean;
    procedure HDel(const AKey, AField: string);

    // List Commands
    procedure RPush(const AKey, AValue: string);
    procedure LPush(const AKey, AValue: string);
    function LPop(const AKey: string): string;
    function LLen(const AKey: string): Integer;
    function LRange(const AKey: string; AStart, AStop: Integer): TArray<string>;

    // Set Commands
    procedure SAdd(const AKey, AMember: string);
    function SMembers(const AKey: string): TArray<string>;
    procedure SRem(const AKey, AMember: string);
    function SIsMember(const AKey, AMember: string): Boolean;

    // Atomic Operations
    function Incr(const AKey: string): Int64;
    function IncrBy(const AKey: string; AAmount: Int64): Int64;
    function Decr(const AKey: string): Int64;

    // Pipeline (Batch Commands)
    function Pipeline(const ACommands: TArray<TArray<string>>): TArray<string>;

    property Connected: Boolean read FConnected;
  end;

  TRedisCache = class(TInterfacedObject, ICache)
  private
    FClient: TRedisClient;
    FKeyPrefix: string;
    FDefaultExpiry: Integer;
    FSerializer: TJsonSerializer;

    function BuildKey(const AKey: string): string;
    function BuildTagKey(const ATag: string): string;

  public
    constructor Create(const AClient: TRedisClient;
      const AKeyPrefix: string = 'app:';
      ADefaultExpiry: Integer = 3600);
    destructor Destroy; override;

    function Get(const AKey: string): string;
    function GetObject<T: class>(const AKey: string): T;
    function TryGet<T>(const AKey: string; out AValue: T): Boolean;
    procedure Set(const AKey, AValue: string; AExpirySeconds: Integer);
    procedure SetObject<T: class>(const AKey: string; const AValue: T;
      AExpirySeconds: Integer);
    procedure Delete(const AKey: string);
    procedure DeleteByPattern(const APattern: string);
    procedure DeleteByTag(const ATag: string);
    function Exists(const AKey: string): Boolean;
    function GetOrCreate<T>(const AKey: string;
      const AFactory: TFunc<T>; AExpirySeconds: Integer): T;
    procedure Invalidate(const ATags: TArray<string>);
    procedure Flush;
  end;

  // Distributed Lock using Redis
  TRedisLock = class
  private
    FClient: TRedisClient;
    FLockKey: string;
    FLockValue: string;
    FAcquired: Boolean;

  public
    constructor Create(const AClient: TRedisClient; const ALockKey: string);
    destructor Destroy; override;

    function TryAcquire(ATimeoutMs: Integer = 5000;
      AExpirySeconds: Integer = 30): Boolean;
    procedure Release;
    function Extend(AExpirySeconds: Integer = 30): Boolean;

    property Acquired: Boolean read FAcquired;
  end;

implementation

// Redis Cache Implementation
function TRedisCache.BuildKey(const AKey: string): string;
begin
  Result := FKeyPrefix + AKey;
end;

function TRedisCache.Get(const AKey: string): string;
begin
  Result := FClient.Get(BuildKey(AKey));
end;

function TRedisCache.TryGet<T>(const AKey: string; out AValue: T): Boolean;
var
  JsonStr: string;
begin
  JsonStr := FClient.Get(BuildKey(AKey));
  if JsonStr = '' then
  begin
    Result := False;
    AValue := Default(T);
    Exit;
  end;

  try
    AValue := FSerializer.Deserialize<T>(JsonStr);
    Result := True;
  except
    AValue := Default(T);
    Result := False;
  end;
end;

procedure TRedisCache.SetObject<T>(const AKey: string; const AValue: T;
  AExpirySeconds: Integer);
var
  JsonStr: string;
  Expiry: Integer;
begin
  JsonStr := FSerializer.Serialize(AValue);
  Expiry := IfThen(AExpirySeconds > 0, AExpirySeconds, FDefaultExpiry);

  if Expiry > 0 then
    FClient.SetEx(BuildKey(AKey), JsonStr, Expiry)
  else
    FClient.Set_(BuildKey(AKey), JsonStr);
end;

function TRedisCache.GetOrCreate<T>(const AKey: string;
  const AFactory: TFunc<T>; AExpirySeconds: Integer): T;
begin
  if not TryGet<T>(AKey, Result) then
  begin
    Result := AFactory();
    if Assigned(TObject(Result)) then
      SetObject<T>(AKey, Result, AExpirySeconds);
  end;
end;

procedure TRedisCache.DeleteByPattern(const APattern: string);
var
  Keys: TArray<string>;
  Key: string;
begin
  Keys := FClient.Keys(FKeyPrefix + APattern);
  for Key in Keys do
    FClient.Del(Key);
end;

procedure TRedisCache.Invalidate(const ATags: TArray<string>);
var
  Tag: string;
begin
  for Tag in ATags do
    DeleteByTag(Tag);
end;

// Distributed Lock
constructor TRedisLock.Create(const AClient: TRedisClient;
  const ALockKey: string);
begin
  inherited Create;
  FClient := AClient;
  FLockKey := 'lock:' + ALockKey;
  FLockValue := TGuid.NewGuid.ToString;
  FAcquired := False;
end;

destructor TRedisLock.Destroy;
begin
  if FAcquired then
    Release;
  inherited Destroy;
end;

function TRedisLock.TryAcquire(ATimeoutMs, AExpirySeconds: Integer): Boolean;
var
  StartTime: TDateTime;
  ElapsedMs: Int64;
begin
  StartTime := Now;
  Result := False;

  repeat
    // SET lock_key value NX PX expiry_ms
    // NX = Only set if not exists
    var LockSet := FClient.SendCommand([
      'SET', FLockKey, FLockValue,
      'NX', 'PX', IntToStr(AExpirySeconds * 1000)
    ]);

    if LockSet = 'OK' then
    begin
      FAcquired := True;
      Result := True;
      Exit;
    end;

    ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
    if ElapsedMs >= ATimeoutMs then Exit;

    Sleep(50); // Wait 50ms before retry
    ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
  until ElapsedMs >= ATimeoutMs;
end;

procedure TRedisLock.Release;
var
  Script: string;
begin
  if not FAcquired then Exit;

  // Lua script to atomic check-and-delete
  Script :=
    'if redis.call("get", KEYS[1]) == ARGV[1] then ' +
    '  return redis.call("del", KEYS[1]) ' +
    'else ' +
    '  return 0 ' +
    'end';

  FClient.SendCommand(['EVAL', Script, '1', FLockKey, FLockValue]);
  FAcquired := False;
end;

end.
```

## 4. Database Sharding

```pascal
// uDatabaseSharding.pas - Horizontal Database Partitioning
unit uDatabaseSharding;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, SyncObjs,
  uConnectionPool;

type
  TShardStrategy = (ssHash, ssRange, ssDirectory, ssGeographic);

  TShardInfo = class
  public
    ShardId: Integer;
    Name: string;
    ConnectionString: string;
    Region: string;
    Weight: Integer;
    IsReadOnly: Boolean;
    Pool: TConnectionPool;
  end;

  IShardRouter = interface
    function GetShardForKey(const AKey: string): TShardInfo;
    function GetShardForId(AId: Int64): TShardInfo;
    function GetAllShards: TArray<TShardInfo>;
    function GetWriteShards: TArray<TShardInfo>;
    function GetReadShards: TArray<TShardInfo>;
  end;

  // Hash-based Sharding
  THashShardRouter = class(TInterfacedObject, IShardRouter)
  private
    FShards: TObjectList<TShardInfo>;
    FShardCount: Integer;

    function HashKey(const AKey: string): Integer;

  public
    constructor Create;
    destructor Destroy; override;

    procedure AddShard(const AConnectionString: string;
      const AName: string = '');

    function GetShardForKey(const AKey: string): TShardInfo;
    function GetShardForId(AId: Int64): TShardInfo;
    function GetAllShards: TArray<TShardInfo>;
    function GetWriteShards: TArray<TShardInfo>;
    function GetReadShards: TArray<TShardInfo>;
  end;

  // Range-based Sharding
  TRangeShardRouter = class(TInterfacedObject, IShardRouter)
  private
    FRanges: TList<record MinId, MaxId: Int64; Shard: TShardInfo; end>;

  public
    procedure AddRange(AMinId, AMaxId: Int64;
      const AConnectionString: string);

    function GetShardForKey(const AKey: string): TShardInfo;
    function GetShardForId(AId: Int64): TShardInfo;
    function GetAllShards: TArray<TShardInfo>;
    function GetWriteShards: TArray<TShardInfo>;
    function GetReadShards: TArray<TShardInfo>;
  end;

  // Sharded Repository Base
  TShardedRepository<T> = class
  private
    FShardRouter: IShardRouter;

    function GetShardForEntity(const AEntity: T): TShardInfo;
    function GetShardKeyFromEntity(const AEntity: T): string;
    function GetShardKeyFromId(AId: Int64): string;

  protected
    function ExecuteOnShard(const AShardInfo: TShardInfo;
      const AQuery: string;
      const AParams: TStringList = nil): TSQLQuery;
    function ExecuteOnAllShards(const AQuery: string;
      const AParams: TStringList = nil): TList<TSQLQuery>;

  public
    constructor Create(const AShardRouter: IShardRouter);

    function GetById(AId: Int64): T;
    procedure Save(const AEntity: T);
    function FindAll(const AWhereClause: string = ''): TList<T>;
    function Count: Int64; // Sum across all shards
  end;

implementation

function THashShardRouter.HashKey(const AKey: string): Integer;
var
  Hash: Cardinal;
  I: Integer;
  C: Char;
begin
  // FNV-1a hash
  Hash := 2166136261;
  for C in AKey do
  begin
    Hash := Hash xor Ord(C);
    Hash := Hash * 16777619;
  end;
  Result := Hash mod Cardinal(FShardCount);
end;

function THashShardRouter.GetShardForKey(const AKey: string): TShardInfo;
var
  ShardIndex: Integer;
begin
  ShardIndex := HashKey(AKey);
  Result := FShards[ShardIndex];
end;

function THashShardRouter.GetShardForId(AId: Int64): TShardInfo;
begin
  Result := GetShardForKey(IntToStr(AId));
end;

end.
```

## 5. Async Processing with Thread Pools

```pascal
// uThreadPool.pas - Thread Pool for Async Processing
unit uThreadPool;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections;

type
  TWorkItem = class
  public
    Work: TProc;
    Callback: TProc<Exception>;
    Priority: Integer;
    SubmitTime: TDateTime;
  end;

  TWorkerThread = class(TThread)
  private
    FPool: TObject; // TThreadPool (forward reference)
    FCurrentWork: TWorkItem;
    FIdleEvent: TEvent;

  protected
    procedure Execute; override;

  public
    constructor Create(const APool: TObject);
    destructor Destroy; override;

    property IsIdle: Boolean read (not Assigned(FCurrentWork));
  end;

  TThreadPoolStats = record
    ActiveWorkers: Integer;
    IdleWorkers: Integer;
    QueuedItems: Integer;
    CompletedItems: Int64;
    FailedItems: Int64;
    AverageExecutionTimeMs: Double;
  end;

  TThreadPool = class
  private
    FWorkers: TObjectList<TWorkerThread>;
    FWorkQueue: TObjectList<TWorkItem>;
    FCriticalSection: TCriticalSection;
    FWorkAvailable: TEvent;
    FMinWorkers: Integer;
    FMaxWorkers: Integer;
    FQueueCapacity: Integer;
    FIsShuttingDown: Boolean;
    FStats: TThreadPoolStats;

    procedure EnsureMinWorkers;
    procedure ScaleIfNeeded;
    procedure WorkerCompleted(const AWorker: TWorkerThread;
      const AError: Exception);

  public
    constructor Create(AMinWorkers: Integer = 2;
      AMaxWorkers: Integer = 20;
      AQueueCapacity: Integer = 1000);
    destructor Destroy; override;

    // Submit work
    procedure Submit(const AWork: TProc;
      const ACallback: TProc<Exception> = nil;
      APriority: Integer = 0);

    function SubmitAndWait<T>(const AWork: TFunc<T>;
      ATimeoutMs: Integer = 30000): T;

    function GetStats: TThreadPoolStats;
    procedure Shutdown(AWaitMs: Integer = 5000);

    class var Default: TThreadPool;
    class constructor ClassCreate;
    class destructor ClassDestroy;
  end;

  // Async/Await Pattern
  TFuture<T> = class
  private
    FValue: T;
    FError: Exception;
    FCompleted: Boolean;
    FEvent: TEvent;
    FCriticalSection: TCriticalSection;

  public
    constructor Create;
    destructor Destroy; override;

    procedure Complete(const AValue: T);
    procedure Fail(const AError: Exception);

    function Await(ATimeoutMs: Integer = 30000): T;
    function TryGetValue(out AValue: T): Boolean;

    function Then_<TResult>(const ANext: TFunc<T, TResult>): TFuture<TResult>;
    function Catch(const AHandler: TProc<Exception>): TFuture<T>;

    property IsCompleted: Boolean read FCompleted;
  end;

implementation

constructor TWorkerThread.Create(const APool: TObject);
begin
  inherited Create(True);
  FPool := APool;
  FIdleEvent := TEvent.Create(nil, False, False, '');
  FreeOnTerminate := False;
end;

destructor TWorkerThread.Destroy;
begin
  FIdleEvent.Free;
  inherited Destroy;
end;

procedure TWorkerThread.Execute;
var
  Pool: TThreadPool;
  WorkItem: TWorkItem;
  StartTime: TDateTime;
  ElapsedMs: Int64;
begin
  Pool := TThreadPool(FPool);

  while not Terminated do
  begin
    WorkItem := nil;

    // Get work from queue
    Pool.FCriticalSection.Acquire;
    try
      if Pool.FWorkQueue.Count > 0 then
      begin
        WorkItem := Pool.FWorkQueue[0];
        Pool.FWorkQueue.Delete(0);
        FCurrentWork := WorkItem;
      end;
    finally
      Pool.FCriticalSection.Release;
    end;

    if Assigned(WorkItem) then
    begin
      StartTime := Now;
      try
        WorkItem.Work();
        if Assigned(WorkItem.Callback) then
          WorkItem.Callback(nil);

        ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
        Pool.FCriticalSection.Acquire;
        try
          Inc(Pool.FStats.CompletedItems);
          Pool.FStats.AverageExecutionTimeMs :=
            (Pool.FStats.AverageExecutionTimeMs + ElapsedMs) / 2;
        finally
          Pool.FCriticalSection.Release;
        end;
      except
        on E: Exception do
        begin
          Inc(Pool.FStats.FailedItems);
          if Assigned(WorkItem.Callback) then
            WorkItem.Callback(E);
        end;
      end;
      FCurrentWork := nil;
      WorkItem.Free;
    end
    else
      Pool.FWorkAvailable.WaitFor(1000);
  end;
end;

constructor TThreadPool.Create(AMinWorkers, AMaxWorkers, AQueueCapacity: Integer);
begin
  inherited Create;
  FMinWorkers := AMinWorkers;
  FMaxWorkers := AMaxWorkers;
  FQueueCapacity := AQueueCapacity;
  FIsShuttingDown := False;

  FCriticalSection := TCriticalSection.Create;
  FWorkAvailable := TEvent.Create(nil, True, False, '');
  FWorkers := TObjectList<TWorkerThread>.Create(True);
  FWorkQueue := TObjectList<TWorkItem>.Create(True);

  EnsureMinWorkers;
end;

destructor TThreadPool.Destroy;
begin
  Shutdown;
  FWorkQueue.Free;
  FWorkers.Free;
  FWorkAvailable.Free;
  FCriticalSection.Free;
  inherited Destroy;
end;

procedure TThreadPool.Submit(const AWork: TProc;
  const ACallback: TProc<Exception>; APriority: Integer);
var
  Item: TWorkItem;
begin
  if FIsShuttingDown then
    raise Exception.Create('Thread pool is shutting down');

  FCriticalSection.Acquire;
  try
    if FWorkQueue.Count >= FQueueCapacity then
      raise EQueueFullException.CreateFmt(
        'Work queue is full (%d items)', [FQueueCapacity]);

    Item := TWorkItem.Create;
    Item.Work := AWork;
    Item.Callback := ACallback;
    Item.Priority := APriority;
    Item.SubmitTime := Now;
    FWorkQueue.Add(Item);
    Inc(FStats.QueuedItems);
  finally
    FCriticalSection.Release;
  end;

  FWorkAvailable.SetEvent;
  ScaleIfNeeded;
end;

function TThreadPool.SubmitAndWait<T>(const AWork: TFunc<T>;
  ATimeoutMs: Integer): T;
var
  Future: TFuture<T>;
begin
  Future := TFuture<T>.Create;
  try
    Submit(
      procedure
      begin
        try
          Future.Complete(AWork());
        except
          on E: Exception do
            Future.Fail(E);
        end;
      end
    );
    Result := Future.Await(ATimeoutMs);
  finally
    Future.Free;
  end;
end;

class constructor TThreadPool.ClassCreate;
begin
  Default := TThreadPool.Create(4, 20, 1000);
end;

class destructor TThreadPool.ClassDestroy;
begin
  Default.Free;
end;

// Future<T> Implementation
constructor TFuture<T>.Create;
begin
  inherited Create;
  FCompleted := False;
  FEvent := TEvent.Create(nil, True, False, '');
  FCriticalSection := TCriticalSection.Create;
end;

destructor TFuture<T>.Destroy;
begin
  FCriticalSection.Free;
  FEvent.Free;
  inherited Destroy;
end;

procedure TFuture<T>.Complete(const AValue: T);
begin
  FCriticalSection.Acquire;
  try
    FValue := AValue;
    FCompleted := True;
    FEvent.SetEvent;
  finally
    FCriticalSection.Release;
  end;
end;

function TFuture<T>.Await(ATimeoutMs: Integer): T;
begin
  if not FCompleted then
  begin
    if FEvent.WaitFor(ATimeoutMs) <> wrSignaled then
      raise EFutureTimeoutException.CreateFmt(
        'Future timed out after %dms', [ATimeoutMs]);
  end;

  if Assigned(FError) then
    raise FError;

  Result := FValue;
end;

end.
```

## 6. Queue-Based Architecture

```pascal
// uQueueProcessor.pas - Background Queue Processor
unit uQueueProcessor;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, SyncObjs,
  uMessageQueue, uThreadPool, uLogger;

type
  TQueueProcessorConfig = record
    QueueName: string;
    ConcurrentWorkers: Integer;
    MaxRetries: Integer;
    RetryDelayMs: Integer;
    BatchSize: Integer;
    PollIntervalMs: Integer;
    AckTimeout: Integer;
  end;

  TMessageHandler = procedure(const AMessage: string;
    const AContext: TJSONObject) of object;

  TQueueProcessor = class
  private
    FConfig: TQueueProcessorConfig;
    FMessageBus: IMessageBus;
    FThreadPool: TThreadPool;
    FLogger: ILogger;
    FHandlers: TDictionary<string, TMessageHandler>;
    FRunning: Boolean;
    FProcessedCount: Int64;
    FFailedCount: Int64;
    FDeadLetterQueue: string;

    procedure ProcessMessage(const ARawMessage: string);
    procedure HandleMessage(const AMessageType: string;
      const APayload: TJSONObject);
    procedure SendToDeadLetter(const AMessage: string;
      const AError: Exception);

  public
    constructor Create(
      const AConfig: TQueueProcessorConfig;
      const AMessageBus: IMessageBus;
      const AThreadPool: TThreadPool);
    destructor Destroy; override;

    procedure RegisterHandler(const AMessageType: string;
      const AHandler: TMessageHandler);
    procedure Start;
    procedure Stop;

    property ProcessedCount: Int64 read FProcessedCount;
    property FailedCount: Int64 read FFailedCount;
  end;

implementation

procedure TQueueProcessor.ProcessMessage(const ARawMessage: string);
var
  MessageJson: TJSONObject;
  MessageType: string;
  Payload: TJSONObject;
  RetryCount: Integer;
  RetryDelay: Integer;
begin
  try
    MessageJson := TJSONObject(GetJSON(ARawMessage));
    try
      MessageType := MessageJson.Get('type', '');
      Payload := TJSONObject(MessageJson.Find('payload'));

      if MessageType = '' then
      begin
        FLogger.Warning('Received message without type: ' + ARawMessage);
        Exit;
      end;

      RetryCount := 0;
      RetryDelay := FConfig.RetryDelayMs;

      repeat
        try
          HandleMessage(MessageType, Payload);
          Inc(FProcessedCount);
          Exit;
        except
          on E: Exception do
          begin
            Inc(RetryCount);
            FLogger.Warning(Format('Message processing failed (attempt %d/%d): %s',
              [RetryCount, FConfig.MaxRetries, E.Message]));

            if RetryCount >= FConfig.MaxRetries then
            begin
              Inc(FFailedCount);
              SendToDeadLetter(ARawMessage, E);
              Exit;
            end;

            // Exponential backoff
            Sleep(RetryDelay);
            RetryDelay := RetryDelay * 2;
          end;
        end;
      until False;
    finally
      MessageJson.Free;
    end;
  except
    on E: Exception do
    begin
      FLogger.Error('Failed to parse message', E);
      Inc(FFailedCount);
    end;
  end;
end;

procedure TQueueProcessor.HandleMessage(const AMessageType: string;
  const APayload: TJSONObject);
var
  Handler: TMessageHandler;
begin
  if not FHandlers.TryGetValue(AMessageType, Handler) then
  begin
    FLogger.Warning('No handler for message type: ' + AMessageType);
    Exit;
  end;

  Handler(AMessageType, APayload);
end;

procedure TQueueProcessor.Start;
begin
  FRunning := True;
  FLogger.Info(Format('Queue processor starting for queue: %s (%d workers)',
    [FConfig.QueueName, FConfig.ConcurrentWorkers]));

  // Subscribe to queue
  for var I := 0 to FConfig.ConcurrentWorkers - 1 do
  begin
    FMessageBus.Subscribe(FConfig.QueueName,
      procedure(AMessage: string)
      begin
        FThreadPool.Submit(
          procedure
          begin
            ProcessMessage(AMessage);
          end
        );
      end
    );
  end;
end;

end.
```

## 7. Load Testing Example

```pascal
// LoadTest/uLoadTester.pas - Simple Load Testing Tool
unit uLoadTester;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections,
  fphttpclient;

type
  TLoadTestResult = record
    TotalRequests: Integer;
    SuccessfulRequests: Integer;
    FailedRequests: Integer;
    AverageResponseTimeMs: Double;
    MinResponseTimeMs: Double;
    MaxResponseTimeMs: Double;
    Percentile95Ms: Double;
    Percentile99Ms: Double;
    RequestsPerSecond: Double;
    TotalDurationSeconds: Double;
    ErrorMessages: TStringList;
  end;

  TLoadTester = class
  private
    FUrl: string;
    FConcurrentUsers: Integer;
    FRequestsPerUser: Integer;
    FResponseTimes: TList<Double>;
    FCriticalSection: TCriticalSection;
    FSuccessCount: Integer;
    FFailCount: Integer;

    procedure RunUserSession(const AUserId: Integer);
    function CalculatePercentile(const ATimes: TList<Double>;
      APercent: Double): Double;

  public
    constructor Create(const AUrl: string;
      AConcurrentUsers: Integer = 10;
      ARequestsPerUser: Integer = 100);
    destructor Destroy; override;

    function Run: TLoadTestResult;
    procedure PrintResults(const AResult: TLoadTestResult);
  end;

implementation

procedure TLoadTester.RunUserSession(const AUserId: Integer);
var
  Client: TFPHTTPClient;
  I: Integer;
  StartTime: TDateTime;
  ElapsedMs: Double;
begin
  Client := TFPHTTPClient.Create(nil);
  try
    Client.ConnectTimeout := 5000;
    Client.IOTimeout := 10000;

    for I := 1 to FRequestsPerUser do
    begin
      StartTime := Now;
      try
        var Response := TStringStream.Create;
        try
          Client.Get(FUrl, Response);
          if Client.ResponseStatusCode < 400 then
          begin
            ElapsedMs := (Now - StartTime) * MSecsPerDay;
            FCriticalSection.Acquire;
            try
              FResponseTimes.Add(ElapsedMs);
              Inc(FSuccessCount);
            finally
              FCriticalSection.Release;
            end;
          end
          else
            Inc(FFailCount);
        finally
          Response.Free;
        end;
      except
        FCriticalSection.Acquire;
        try
          Inc(FFailCount);
        finally
          FCriticalSection.Release;
        end;
      end;
    end;
  finally
    Client.Free;
  end;
end;

function TLoadTester.Run: TLoadTestResult;
var
  Threads: array of TThread;
  I: Integer;
  StartTime: TDateTime;
  SortedTimes: TList<Double>;
  ElapsedMs: Double;
begin
  SetLength(Threads, FConcurrentUsers);
  FSuccessCount := 0;
  FFailCount := 0;
  FResponseTimes.Clear;

  StartTime := Now;

  for I := 0 to FConcurrentUsers - 1 do
  begin
    var UserId := I;
    Threads[I] := TThread.CreateAnonymousThread(
      procedure
      begin
        RunUserSession(UserId);
      end
    );
    Threads[I].FreeOnTerminate := False;
    Threads[I].Start;
  end;

  // Wait for all threads
  for I := 0 to FConcurrentUsers - 1 do
  begin
    Threads[I].WaitFor;
    Threads[I].Free;
  end;

  // Calculate results
  Result.TotalDurationSeconds := (Now - StartTime) * SecsPerDay;
  Result.TotalRequests := FConcurrentUsers * FRequestsPerUser;
  Result.SuccessfulRequests := FSuccessCount;
  Result.FailedRequests := FFailCount;

  if FResponseTimes.Count > 0 then
  begin
    SortedTimes := TList<Double>.Create;
    try
      SortedTimes.AddRange(FResponseTimes);
      SortedTimes.Sort;

      Result.MinResponseTimeMs := SortedTimes[0];
      Result.MaxResponseTimeMs := SortedTimes[SortedTimes.Count - 1];

      var Total: Double := 0;
      for ElapsedMs in FResponseTimes do
        Total := Total + ElapsedMs;
      Result.AverageResponseTimeMs := Total / FResponseTimes.Count;

      Result.Percentile95Ms := CalculatePercentile(SortedTimes, 95);
      Result.Percentile99Ms := CalculatePercentile(SortedTimes, 99);
    finally
      SortedTimes.Free;
    end;
  end;

  if Result.TotalDurationSeconds > 0 then
    Result.RequestsPerSecond :=
      Result.SuccessfulRequests / Result.TotalDurationSeconds;
end;

procedure TLoadTester.PrintResults(const AResult: TLoadTestResult);
begin
  WriteLn(#13#10'=== Load Test Results ===');
  WriteLn(Format('URL: %s', [FUrl]));
  WriteLn(Format('Concurrent Users: %d', [FConcurrentUsers]));
  WriteLn(Format('Requests per User: %d', [FRequestsPerUser]));
  WriteLn('');
  WriteLn(Format('Total Requests: %d', [AResult.TotalRequests]));
  WriteLn(Format('Successful: %d (%.1f%%)',
    [AResult.SuccessfulRequests,
     (AResult.SuccessfulRequests / AResult.TotalRequests) * 100]));
  WriteLn(Format('Failed: %d', [AResult.FailedRequests]));
  WriteLn('');
  WriteLn(Format('Duration: %.2f seconds', [AResult.TotalDurationSeconds]));
  WriteLn(Format('Throughput: %.2f req/s', [AResult.RequestsPerSecond]));
  WriteLn('');
  WriteLn(Format('Response Times:'));
  WriteLn(Format('  Min: %.1f ms', [AResult.MinResponseTimeMs]));
  WriteLn(Format('  Avg: %.1f ms', [AResult.AverageResponseTimeMs]));
  WriteLn(Format('  Max: %.1f ms', [AResult.MaxResponseTimeMs]));
  WriteLn(Format('  P95: %.1f ms', [AResult.Percentile95Ms]));
  WriteLn(Format('  P99: %.1f ms', [AResult.Percentile99Ms]));
end;

end.
```

## 8. สรุปกลยุทธ์ Scalability

| กลยุทธ์ | เมื่อใช้ | ประโยชน์ |
|---------|---------|---------|
| Connection Pooling | ทุก DB-intensive app | ลด overhead การเชื่อมต่อ |
| Redis Cache | Read-heavy data | ลด DB load ได้ 10-100x |
| DB Sharding | ข้อมูลมาก (>1TB) | Scale Database horizontally |
| Thread Pool | CPU-bound tasks | ใช้ CPU cores ได้เต็มที่ |
| Message Queue | Async processing | Decouple services |
| Load Balancer | Multiple instances | Distribute load |

**กฎ Scalability ที่สำคัญ:**
1. **Cache first** - ก่อน Scale DB ลอง Cache ก่อน
2. **Measure first** - Profile ก่อน Optimize
3. **Stateless** - Design service ให้ Stateless เพื่อ Scale ได้ง่าย
4. **Async** - งาน Background ควรเป็น Async
5. **Database** - มักเป็น Bottleneck หลัก
