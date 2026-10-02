# ตอนที่ 87: High Availability กับ Pascal/Lazarus

## บทนำ: ระบบ High Availability

High Availability (HA) คือการออกแบบระบบให้ทำงานต่อเนื่องโดยมี Downtime น้อยที่สุด เป้าหมายคือ "Five Nines" (99.999% Uptime = ~5 นาที Downtime/ปี)

```
Availability Levels:
99%     = ~87.6 ชั่วโมง downtime/ปี
99.9%   = ~8.76 ชั่วโมง downtime/ปี
99.99%  = ~52.6 นาที downtime/ปี
99.999% = ~5.26 นาที downtime/ปี
```

## 1. Circuit Breaker Pattern

```pascal
// uCircuitBreaker.pas - Circuit Breaker Pattern
unit uCircuitBreaker;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections;

type
  TCircuitBreakerState = (cbsClosed, cbsOpen, cbsHalfOpen);

  TCircuitBreakerConfig = record
    FailureThreshold: Integer;     // จำนวนครั้งที่ Fail ก่อน Open
    SuccessThreshold: Integer;     // จำนวนครั้งที่ Success ก่อน Close
    Timeout: Integer;              // ms ก่อน Half-Open
    SamplingWindow: Integer;       // seconds สำหรับนับ Failures
    MinimumRequests: Integer;      // requests ขั้นต่ำก่อนพิจารณา Open
  end;

  TCircuitBreakerEvent = (cbeOpened, cbeClosed, cbeHalfOpened, cbeSucceeded, cbeFailed);
  TCircuitBreakerEventHandler = procedure(const AName: string;
    AEvent: TCircuitBreakerEvent) of object;

  TCircuitBreaker = class
  private
    FName: string;
    FConfig: TCircuitBreakerConfig;
    FState: TCircuitBreakerState;
    FFailureCount: Integer;
    FSuccessCount: Integer;
    FRequestCount: Integer;
    FLastFailureTime: TDateTime;
    FStateChangedAt: TDateTime;
    FWindowStart: TDateTime;
    FEventHandlers: TList<TCircuitBreakerEventHandler>;
    FCriticalSection: TCriticalSection;
    FLogger: ILogger;

    procedure RecordSuccess;
    procedure RecordFailure;
    procedure TransitionTo(ANewState: TCircuitBreakerState);
    procedure ResetCounts;
    procedure NotifyEvent(AEvent: TCircuitBreakerEvent);
    function ShouldOpen: Boolean;
    function ShouldClose: Boolean;
    function IsTimeoutExpired: Boolean;

  public
    constructor Create(const AName: string;
      const AConfig: TCircuitBreakerConfig);
    destructor Destroy; override;

    function Execute<T>(const AFunc: TFunc<T>): T;
    procedure Execute(const AAction: TProc);
    function CanAttempt: Boolean;
    procedure Reset;

    procedure AddEventHandler(const AHandler: TCircuitBreakerEventHandler);

    property Name: string read FName;
    property State: TCircuitBreakerState read FState;
    property FailureCount: Integer read FFailureCount;
  end;

  ECircuitBreakerOpenException = class(Exception);

  // Circuit Breaker Registry
  TCircuitBreakerRegistry = class
  private
    FBreakers: TObjectDictionary<string, TCircuitBreaker>;
    FCriticalSection: TCriticalSection;
    FDefaultConfig: TCircuitBreakerConfig;

    class var FInstance: TCircuitBreakerRegistry;

  public
    constructor Create;
    destructor Destroy; override;

    function GetOrCreate(const AName: string;
      const AConfig: TCircuitBreakerConfig): TCircuitBreaker;
    function GetBreaker(const AName: string): TCircuitBreaker;
    function GetAllBreakers: TArray<TCircuitBreaker>;
    procedure ResetAll;

    class function Instance: TCircuitBreakerRegistry;
    class constructor ClassCreate;
    class destructor ClassDestroy;
  end;

implementation

constructor TCircuitBreaker.Create(const AName: string;
  const AConfig: TCircuitBreakerConfig);
begin
  inherited Create;
  FName := AName;
  FConfig := AConfig;
  FState := cbsClosed;
  FCriticalSection := TCriticalSection.Create;
  FEventHandlers := TList<TCircuitBreakerEventHandler>.Create;
  FWindowStart := Now;
  FStateChangedAt := Now;
  ResetCounts;
end;

destructor TCircuitBreaker.Destroy;
begin
  FEventHandlers.Free;
  FCriticalSection.Free;
  inherited Destroy;
end;

function TCircuitBreaker.Execute<T>(const AFunc: TFunc<T>): T;
begin
  if not CanAttempt then
    raise ECircuitBreakerOpenException.CreateFmt(
      'Circuit breaker "%s" is OPEN', [FName]);

  try
    Result := AFunc();
    RecordSuccess;
  except
    on E: ECircuitBreakerOpenException do raise;
    on E: Exception do
    begin
      RecordFailure;
      raise;
    end;
  end;
end;

function TCircuitBreaker.CanAttempt: Boolean;
begin
  FCriticalSection.Acquire;
  try
    case FState of
      cbsClosed:
        Result := True;

      cbsOpen:
      begin
        if IsTimeoutExpired then
        begin
          TransitionTo(cbsHalfOpen);
          Result := True;
        end
        else
          Result := False;
      end;

      cbsHalfOpen:
        Result := True;
    end;
  finally
    FCriticalSection.Release;
  end;
end;

procedure TCircuitBreaker.RecordSuccess;
begin
  FCriticalSection.Acquire;
  try
    Inc(FSuccessCount);
    Inc(FRequestCount);

    case FState of
      cbsHalfOpen:
        if ShouldClose then
          TransitionTo(cbsClosed);
      cbsClosed:
        ; // Normal operation
    end;

    NotifyEvent(cbeSucceeded);
  finally
    FCriticalSection.Release;
  end;
end;

procedure TCircuitBreaker.RecordFailure;
begin
  FCriticalSection.Acquire;
  try
    FLastFailureTime := Now;
    Inc(FFailureCount);
    Inc(FRequestCount);

    // Reset window if expired
    if (Now - FWindowStart) * SecsPerDay > FConfig.SamplingWindow then
    begin
      FWindowStart := Now;
      FFailureCount := 1;
      FRequestCount := 1;
      FSuccessCount := 0;
    end;

    case FState of
      cbsClosed:
        if ShouldOpen then
          TransitionTo(cbsOpen);

      cbsHalfOpen:
        TransitionTo(cbsOpen); // Any failure in half-open goes back to open
    end;

    NotifyEvent(cbeFailed);
  finally
    FCriticalSection.Release;
  end;
end;

function TCircuitBreaker.ShouldOpen: Boolean;
begin
  Result := (FRequestCount >= FConfig.MinimumRequests) and
    (FFailureCount >= FConfig.FailureThreshold);
end;

function TCircuitBreaker.ShouldClose: Boolean;
begin
  Result := FSuccessCount >= FConfig.SuccessThreshold;
end;

function TCircuitBreaker.IsTimeoutExpired: Boolean;
begin
  Result := Round((Now - FStateChangedAt) * MSecsPerDay) >= FConfig.Timeout;
end;

procedure TCircuitBreaker.TransitionTo(ANewState: TCircuitBreakerState);
begin
  FState := ANewState;
  FStateChangedAt := Now;
  ResetCounts;

  case ANewState of
    cbsOpen: NotifyEvent(cbeOpened);
    cbsClosed: NotifyEvent(cbeClosed);
    cbsHalfOpen: NotifyEvent(cbeHalfOpened);
  end;
end;

procedure TCircuitBreaker.ResetCounts;
begin
  FFailureCount := 0;
  FSuccessCount := 0;
  FRequestCount := 0;
end;

procedure TCircuitBreaker.NotifyEvent(AEvent: TCircuitBreakerEvent);
var
  Handler: TCircuitBreakerEventHandler;
begin
  for Handler in FEventHandlers do
    try
      Handler(FName, AEvent);
    except
      // Ignore handler errors
    end;
end;

end.
```

## 2. Retry Pattern with Exponential Backoff

```pascal
// uRetryPolicy.pas - Retry with Exponential Backoff
unit uRetryPolicy;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TRetryCondition = function(const AException: Exception): Boolean;

  TRetryPolicy = class
  private
    FMaxAttempts: Integer;
    FInitialDelayMs: Integer;
    FMaxDelayMs: Integer;
    FBackoffMultiplier: Double;
    FJitterFactor: Double;
    FRetryCondition: TRetryCondition;
    FOnRetry: TProc<Exception, Integer>;
    FLogger: ILogger;

    function ShouldRetry(const AException: Exception): Boolean;
    function CalculateDelay(AAttempt: Integer): Integer;

  public
    constructor Create(
      AMaxAttempts: Integer = 3;
      AInitialDelayMs: Integer = 1000;
      AMaxDelayMs: Integer = 30000;
      ABackoffMultiplier: Double = 2.0);

    function Execute<T>(const AFunc: TFunc<T>): T;
    procedure Execute(const AAction: TProc);

    function WithMaxAttempts(AMax: Integer): TRetryPolicy;
    function WithInitialDelay(ADelayMs: Integer): TRetryPolicy;
    function WithMaxDelay(ADelayMs: Integer): TRetryPolicy;
    function WithBackoffMultiplier(AMultiplier: Double): TRetryPolicy;
    function WithJitter(AFactor: Double = 0.1): TRetryPolicy;
    function WhenException(const ACondition: TRetryCondition): TRetryPolicy;
    function OnRetry(const AHandler: TProc<Exception, Integer>): TRetryPolicy;

    class function Default: TRetryPolicy;
    class function ForNetworkErrors: TRetryPolicy;
    class function ForDatabaseErrors: TRetryPolicy;
  end;

implementation

constructor TRetryPolicy.Create(AMaxAttempts, AInitialDelayMs, AMaxDelayMs: Integer;
  ABackoffMultiplier: Double);
begin
  inherited Create;
  FMaxAttempts := AMaxAttempts;
  FInitialDelayMs := AInitialDelayMs;
  FMaxDelayMs := AMaxDelayMs;
  FBackoffMultiplier := ABackoffMultiplier;
  FJitterFactor := 0;
  FRetryCondition := nil;
end;

function TRetryPolicy.Execute<T>(const AFunc: TFunc<T>): T;
var
  Attempt: Integer;
  Delay: Integer;
  LastException: Exception;
begin
  for Attempt := 1 to FMaxAttempts do
  begin
    try
      Result := AFunc();
      Exit;
    except
      on E: Exception do
      begin
        LastException := E;

        if Attempt >= FMaxAttempts then
          raise;

        if not ShouldRetry(E) then
          raise;

        Delay := CalculateDelay(Attempt);

        if Assigned(FOnRetry) then
          FOnRetry(E, Attempt);

        if Assigned(FLogger) then
          FLogger.Warning(Format('[Retry] Attempt %d/%d failed: %s. Waiting %dms...',
            [Attempt, FMaxAttempts, E.Message, Delay]));

        Sleep(Delay);
      end;
    end;
  end;

  // Should not reach here
  raise LastException;
end;

function TRetryPolicy.CalculateDelay(AAttempt: Integer): Integer;
var
  BaseDelay: Double;
  Jitter: Integer;
begin
  BaseDelay := FInitialDelayMs * Power(FBackoffMultiplier, AAttempt - 1);
  if BaseDelay > FMaxDelayMs then BaseDelay := FMaxDelayMs;

  Result := Round(BaseDelay);

  // Add jitter to prevent thundering herd
  if FJitterFactor > 0 then
  begin
    Jitter := Round(BaseDelay * FJitterFactor * (Random - 0.5) * 2);
    Result := Result + Jitter;
    if Result < 0 then Result := FInitialDelayMs;
  end;
end;

function TRetryPolicy.ShouldRetry(const AException: Exception): Boolean;
begin
  if Assigned(FRetryCondition) then
    Result := FRetryCondition(AException)
  else
    // Default: retry on transient errors
    Result := not (AException is EArgumentException) and
              not (AException is EAuthorizationException);
end;

class function TRetryPolicy.ForNetworkErrors: TRetryPolicy;
begin
  Result := TRetryPolicy.Create(3, 1000, 10000, 2.0);
  Result.WithJitter(0.1);
  Result.WhenException(
    function(AException: Exception): Boolean
    begin
      Result := AException is EHTTPException or
                AException is ESocketException or
                AException is EConnectionException;
    end
  );
end;

class function TRetryPolicy.ForDatabaseErrors: TRetryPolicy;
begin
  Result := TRetryPolicy.Create(5, 500, 5000, 1.5);
  Result.WhenException(
    function(AException: Exception): Boolean
    begin
      Result := AException is EDeadlockException or
                AException is EConnectionException or
                (AException is ESQLException and
                 ESQLException(AException).IsTransient);
    end
  );
end;

end.
```

## 3. Health Check System

```pascal
// uHealthCheck.pas - Comprehensive Health Checks
unit uHealthCheck;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, fpjson,
  SyncObjs, fphttpclient;

type
  THealthStatus = (hsHealthy, hsDegraded, hsUnhealthy);

  THealthCheckResult = record
    Name: string;
    Status: THealthStatus;
    Description: string;
    Duration: Integer; // milliseconds
    Data: TJSONObject;
    CheckedAt: TDateTime;
  end;

  THealthReport = class
  public
    OverallStatus: THealthStatus;
    Checks: TList<THealthCheckResult>;
    Duration: Integer;
    Timestamp: TDateTime;

    constructor Create;
    destructor Destroy; override;

    function ToJson: TJSONObject;
  end;

  IHealthCheck = interface
    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // Database Health Check
  TDatabaseHealthCheck = class(TInterfacedObject, IHealthCheck)
  private
    FName: string;
    FConnectionPool: TConnectionPool;

  public
    constructor Create(const AName: string;
      const APool: TConnectionPool);

    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // Redis Health Check
  TRedisHealthCheck = class(TInterfacedObject, IHealthCheck)
  private
    FClient: TRedisClient;
  public
    constructor Create(const AClient: TRedisClient);
    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // HTTP Endpoint Health Check
  THttpHealthCheck = class(TInterfacedObject, IHealthCheck)
  private
    FName: string;
    FUrl: string;
    FExpectedStatus: Integer;
    FTimeoutMs: Integer;
  public
    constructor Create(const AName, AUrl: string;
      AExpectedStatus: Integer = 200;
      ATimeoutMs: Integer = 5000);
    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // Disk Space Health Check
  TDiskSpaceHealthCheck = class(TInterfacedObject, IHealthCheck)
  private
    FPath: string;
    FMinFreeBytes: Int64;
    FWarningThresholdPct: Double;
  public
    constructor Create(const APath: string;
      AMinFreeBytes: Int64 = 1073741824; // 1GB
      AWarningThresholdPct: Double = 0.2); // 20%
    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // Memory Health Check
  TMemoryHealthCheck = class(TInterfacedObject, IHealthCheck)
  private
    FMaxMemoryMB: Integer;
    FWarningThresholdPct: Double;
  public
    constructor Create(AMaxMemoryMB: Integer = 2048;
      AWarningThresholdPct: Double = 0.8);
    function GetName: string;
    function Check: THealthCheckResult;
  end;

  // Health Check Runner
  THealthCheckService = class
  private
    FChecks: TInterfaceList;
    FLastReport: THealthReport;
    FCriticalSection: TCriticalSection;
    FLogger: ILogger;

  public
    constructor Create;
    destructor Destroy; override;

    procedure RegisterCheck(const ACheck: IHealthCheck);
    function RunChecks: THealthReport;
    function GetLastReport: THealthReport;
    function IsHealthy: Boolean;
  end;

implementation

// Database Health Check
function TDatabaseHealthCheck.Check: THealthCheckResult;
var
  StartTime: TDateTime;
  Conn: TPooledConnection;
  Query: TSQLQuery;
begin
  Result.Name := FName;
  Result.CheckedAt := Now;
  StartTime := Now;

  try
    Conn := FConnectionPool.Acquire;
    try
      Query := TSQLQuery.Create(nil);
      try
        Query.DataBase := Conn.Connection;
        Query.SQL.Text := 'SELECT 1 AS health_check';
        Query.Open;
        Query.Close;

        Result.Status := hsHealthy;
        Result.Description := 'Database connection successful';

        var Stats := FConnectionPool.GetStats;
        Result.Data := TJSONObject.Create;
        Result.Data.Add('activeConnections', Stats.ActiveConnections);
        Result.Data.Add('idleConnections', Stats.IdleConnections);
        Result.Data.Add('totalConnections', Stats.TotalConnections);
      finally
        Query.Free;
      end;
    finally
      FConnectionPool.Release(Conn);
    end;
  except
    on E: Exception do
    begin
      Result.Status := hsUnhealthy;
      Result.Description := 'Database connection failed: ' + E.Message;
    end;
  end;

  Result.Duration := Round((Now - StartTime) * MSecsPerDay);
end;

// HTTP Health Check
function THttpHealthCheck.Check: THealthCheckResult;
var
  Client: TFPHTTPClient;
  StartTime: TDateTime;
  Response: TStringStream;
begin
  Result.Name := FName;
  Result.CheckedAt := Now;
  StartTime := Now;

  Client := TFPHTTPClient.Create(nil);
  Response := TStringStream.Create;
  try
    try
      Client.ConnectTimeout := FTimeoutMs;
      Client.IOTimeout := FTimeoutMs;
      Client.Get(FUrl, Response);

      if Client.ResponseStatusCode = FExpectedStatus then
      begin
        Result.Status := hsHealthy;
        Result.Description := Format('%s returned %d', [FUrl, FExpectedStatus]);
      end
      else
      begin
        Result.Status := hsDegraded;
        Result.Description := Format('%s returned unexpected %d',
          [FUrl, Client.ResponseStatusCode]);
      end;

      Result.Data := TJSONObject.Create;
      Result.Data.Add('statusCode', Client.ResponseStatusCode);
      Result.Data.Add('responseSize', Response.Size);
    except
      on E: Exception do
      begin
        Result.Status := hsUnhealthy;
        Result.Description := Format('Failed to reach %s: %s', [FUrl, E.Message]);
      end;
    end;
  finally
    Response.Free;
    Client.Free;
  end;

  Result.Duration := Round((Now - StartTime) * MSecsPerDay);
end;

// Disk Space Health Check
function TDiskSpaceHealthCheck.Check: THealthCheckResult;
var
  TotalSpace, FreeSpace: Int64;
  FreePercent: Double;
  StartTime: TDateTime;
begin
  Result.Name := 'disk_space';
  Result.CheckedAt := Now;
  StartTime := Now;

  {$IFDEF UNIX}
  var StatFS: TStatFS;
  if FpStatFS(PChar(FPath), @StatFS) = 0 then
  begin
    TotalSpace := StatFS.f_blocks * StatFS.f_bsize;
    FreeSpace := StatFS.f_bfree * StatFS.f_bsize;
  end;
  {$ELSE}
  GetDiskFreeSpaceEx(PChar(FPath), FreeSpace, TotalSpace, nil);
  {$ENDIF}

  if TotalSpace > 0 then
    FreePercent := FreeSpace / TotalSpace
  else
    FreePercent := 0;

  Result.Data := TJSONObject.Create;
  Result.Data.Add('totalSpaceMB', Round(TotalSpace / (1024*1024)));
  Result.Data.Add('freeSpaceMB', Round(FreeSpace / (1024*1024)));
  Result.Data.Add('freePercent', Round(FreePercent * 100));

  if FreeSpace < FMinFreeBytes then
  begin
    Result.Status := hsUnhealthy;
    Result.Description := Format('Insufficient disk space: %.1f GB free',
      [FreeSpace / (1024*1024*1024)]);
  end
  else if FreePercent < FWarningThresholdPct then
  begin
    Result.Status := hsDegraded;
    Result.Description := Format('Low disk space: %.1f%% free', [FreePercent * 100]);
  end
  else
  begin
    Result.Status := hsHealthy;
    Result.Description := Format('Disk space OK: %.1f%% free', [FreePercent * 100]);
  end;

  Result.Duration := Round((Now - StartTime) * MSecsPerDay);
end;

// Health Check Service
function THealthCheckService.RunChecks: THealthReport;
var
  Check: IHealthCheck;
  CheckResult: THealthCheckResult;
  StartTime: TDateTime;
begin
  StartTime := Now;
  Result := THealthReport.Create;
  Result.Timestamp := Now;
  Result.OverallStatus := hsHealthy;

  for var I := 0 to FChecks.Count - 1 do
  begin
    Check := IHealthCheck(FChecks[I]);
    try
      CheckResult := Check.Check;
    except
      on E: Exception do
      begin
        CheckResult.Name := Check.GetName;
        CheckResult.Status := hsUnhealthy;
        CheckResult.Description := 'Check threw exception: ' + E.Message;
        CheckResult.CheckedAt := Now;
      end;
    end;

    Result.Checks.Add(CheckResult);

    // Determine overall status
    case CheckResult.Status of
      hsDegraded:
        if Result.OverallStatus = hsHealthy then
          Result.OverallStatus := hsDegraded;
      hsUnhealthy:
        Result.OverallStatus := hsUnhealthy;
    end;
  end;

  Result.Duration := Round((Now - StartTime) * MSecsPerDay);

  FCriticalSection.Acquire;
  try
    FLastReport.Free;
    FLastReport := Result;
  finally
    FCriticalSection.Release;
  end;
end;

// Health Report to JSON
function THealthReport.ToJson: TJSONObject;
var
  ChecksArray: TJSONArray;
  CheckObj: TJSONObject;
  StatusNames: array[THealthStatus] of string;
  CheckResult: THealthCheckResult;
begin
  StatusNames[hsHealthy] := 'healthy';
  StatusNames[hsDegraded] := 'degraded';
  StatusNames[hsUnhealthy] := 'unhealthy';

  Result := TJSONObject.Create;
  Result.Add('status', StatusNames[OverallStatus]);
  Result.Add('duration', Duration);
  Result.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', Timestamp));

  ChecksArray := TJSONArray.Create;
  for CheckResult in Checks do
  begin
    CheckObj := TJSONObject.Create;
    CheckObj.Add('name', CheckResult.Name);
    CheckObj.Add('status', StatusNames[CheckResult.Status]);
    CheckObj.Add('description', CheckResult.Description);
    CheckObj.Add('duration', CheckResult.Duration);
    if Assigned(CheckResult.Data) then
      CheckObj.Add('data', CheckResult.Data.Clone);
    ChecksArray.Add(CheckObj);
  end;
  Result.Add('checks', ChecksArray);
end;

end.
```

## 4. Graceful Degradation

```pascal
// uGracefulDegradation.pas - Fallback Strategies
unit uGracefulDegradation;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, uCircuitBreaker, uCache;

type
  // Fallback Chain Pattern
  TFallbackChain<T> = class
  private
    FPrimary: TFunc<T>;
    FFallbacks: TList<TFunc<T>>;
    FDefaultValue: T;
    FLogger: ILogger;

  public
    constructor Create(const APrimary: TFunc<T>; const ADefault: T);
    destructor Destroy; override;

    function WithFallback(const AFallback: TFunc<T>): TFallbackChain<T>;
    function Execute: T;
  end;

  // Cache-Aside with Stale Data Fallback
  TCacheAsideWithStaleFallback<T: class> = class
  private
    FCache: ICache;
    FLoader: TFunc<T>;
    FCacheKey: string;
    FTtl: Integer;
    FStaleTtl: Integer; // Time to keep stale data

  public
    constructor Create(const ACache: ICache; const ACacheKey: string;
      const ALoader: TFunc<T>; ATtl: Integer = 300;
      AStaleTtl: Integer = 3600);

    function Get: T;
    procedure Invalidate;
  end;

  // Feature Flags for Graceful Degradation
  TFeatureFlag = class
  private
    FName: string;
    FEnabled: Boolean;
    FRolloutPercentage: Integer;  // 0-100
    FCriticalSection: TCriticalSection;

  public
    constructor Create(const AName: string; AEnabled: Boolean = True;
      ARolloutPercentage: Integer = 100);
    destructor Destroy; override;

    function IsEnabled(const AUserId: string = ''): Boolean;
    procedure Enable;
    procedure Disable;
    procedure SetRollout(APercentage: Integer);

    property Name: string read FName;
  end;

  TFeatureFlagService = class
  private
    FFlags: TObjectDictionary<string, TFeatureFlag>;
    FRemoteUrl: string;
    FLogger: ILogger;
    FCriticalSection: TCriticalSection;

    procedure LoadFromRemote;

  public
    constructor Create(const ARemoteUrl: string = '');
    destructor Destroy; override;

    procedure Register(const AFlag: TFeatureFlag);
    function IsEnabled(const AFlagName: string;
      const AUserId: string = ''): Boolean;
    procedure Override(const AFlagName: string; AEnabled: Boolean);
    procedure Refresh;

    class var Default: TFeatureFlagService;
  end;

implementation

// Fallback Chain
function TFallbackChain<T>.Execute: T;
var
  I: Integer;
  LastError: Exception;
begin
  try
    Result := FPrimary();
    Exit;
  except
    on E: Exception do
    begin
      LastError := E;
      FLogger.Warning('Primary function failed, trying fallbacks: ' + E.Message);
    end;
  end;

  for I := 0 to FFallbacks.Count - 1 do
  begin
    try
      Result := FFallbacks[I]();
      FLogger.Info(Format('Fallback %d succeeded', [I + 1]));
      Exit;
    except
      on E: Exception do
      begin
        LastError := E;
        FLogger.Warning(Format('Fallback %d failed: %s', [I + 1, E.Message]));
      end;
    end;
  end;

  // Return default value instead of raising
  FLogger.Warning('All fallbacks failed, returning default value');
  Result := FDefaultValue;
end;

// Feature Flag
function TFeatureFlag.IsEnabled(const AUserId: string): Boolean;
begin
  FCriticalSection.Acquire;
  try
    if not FEnabled then
    begin
      Result := False;
      Exit;
    end;

    if FRolloutPercentage >= 100 then
    begin
      Result := True;
      Exit;
    end;

    if AUserId <> '' then
    begin
      // Consistent hash for user-based rollout
      var Hash: Cardinal := 2166136261;
      for var C in (FName + ':' + AUserId) do
        Hash := (Hash xor Ord(C)) * 16777619;
      Result := (Hash mod 100) < Cardinal(FRolloutPercentage);
    end
    else
      Result := Random(100) < FRolloutPercentage;
  finally
    FCriticalSection.Release;
  end;
end;

// Bulkhead Pattern - Isolate Resources
type
  TBulkhead = class
  private
    FSemaphore: TSemaphore;
    FMaxConcurrent: Integer;
    FName: string;
    FRejectedCount: Int64;
    FCurrentCount: Integer;
    FCriticalSection: TCriticalSection;

  public
    constructor Create(const AName: string; AMaxConcurrent: Integer = 10);
    destructor Destroy; override;

    function Execute<T>(const AFunc: TFunc<T>;
      ATimeoutMs: Integer = 5000): T;

    property Name: string read FName;
    property CurrentCount: Integer read FCurrentCount;
    property RejectedCount: Int64 read FRejectedCount;
  end;

function TBulkhead.Execute<T>(const AFunc: TFunc<T>; ATimeoutMs: Integer): T;
begin
  if FSemaphore.WaitFor(ATimeoutMs) <> wrSignaled then
  begin
    Inc(FRejectedCount);
    raise EBulkheadRejectedException.CreateFmt(
      'Bulkhead "%s" rejected request (max concurrent: %d)',
      [FName, FMaxConcurrent]);
  end;

  FCriticalSection.Acquire;
  try
    Inc(FCurrentCount);
  finally
    FCriticalSection.Release;
  end;

  try
    Result := AFunc();
  finally
    FSemaphore.Release;
    FCriticalSection.Acquire;
    try
      Dec(FCurrentCount);
    finally
      FCriticalSection.Release;
    end;
  end;
end;

end.
```

## 5. Complete HA System Setup

```pascal
// HaSystem/HaSystem.pas - Complete HA Setup
program HaSystem;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, SyncObjs,
  uCircuitBreaker, uRetryPolicy, uHealthCheck,
  uGracefulDegradation, uConnectionPool,
  uLogger, uConfiguration;

// HA Configuration Setup
procedure SetupHighAvailability;
var
  DbBreaker: TCircuitBreaker;
  PaymentBreaker: TCircuitBreaker;
  Config: TCircuitBreakerConfig;
  HealthService: THealthCheckService;
  DbPool: TConnectionPool;
begin
  // 1. Database Connection Pool
  DbPool := TConnectionPool.Create(
    Config.GetConnectionString,
    5,    // Min connections
    50,   // Max connections
    30000, // Timeout 30s
    600,  // Idle timeout 10min
    3600  // Max lifetime 1hr
  );

  // 2. Circuit Breakers
  Config.FailureThreshold := 5;
  Config.SuccessThreshold := 2;
  Config.Timeout := 60000;
  Config.SamplingWindow := 60;
  Config.MinimumRequests := 10;

  DbBreaker := TCircuitBreakerRegistry.Instance.GetOrCreate('database', Config);
  DbBreaker.AddEventHandler(
    procedure(AName: string; AEvent: TCircuitBreakerEvent)
    begin
      case AEvent of
        cbeOpened:
          FLogger.Warning(Format('Circuit breaker "%s" OPENED', [AName]));
        cbeClosed:
          FLogger.Info(Format('Circuit breaker "%s" CLOSED', [AName]));
      end;
    end
  );

  Config.FailureThreshold := 3;
  Config.Timeout := 30000;
  PaymentBreaker := TCircuitBreakerRegistry.Instance.GetOrCreate('payment', Config);

  // 3. Health Checks
  HealthService := THealthCheckService.Create;
  HealthService.RegisterCheck(
    TDatabaseHealthCheck.Create('main_db', DbPool));
  HealthService.RegisterCheck(
    TRedisHealthCheck.Create(RedisClient));
  HealthService.RegisterCheck(
    THttpHealthCheck.Create('payment_service',
      'http://payment-service/health', 200, 3000));
  HealthService.RegisterCheck(
    TDiskSpaceHealthCheck.Create('/var/data'));
  HealthService.RegisterCheck(
    TMemoryHealthCheck.Create(4096)); // 4GB max

  WriteLn('HA System configured successfully');
  WriteLn('Circuit breakers: 2');
  WriteLn('Health checks: 5');
  WriteLn('Connection pool: min=5, max=50');
end;

// Example: HA Order Processing
procedure ProcessOrderWithHA(AOrderId: Integer);
var
  DbBreaker: TCircuitBreaker;
  RetryPolicy: TRetryPolicy;
  Fallback: TFallbackChain<TOrderDto>;
  Result: TOrderDto;
begin
  DbBreaker := TCircuitBreakerRegistry.Instance.GetBreaker('database');
  RetryPolicy := TRetryPolicy.ForDatabaseErrors;

  try
    // With Circuit Breaker + Retry + Fallback
    Fallback := TFallbackChain<TOrderDto>.Create(
      // Primary: Get from database
      function: TOrderDto
      begin
        Result := DbBreaker.Execute<TOrderDto>(
          function: TOrderDto
          begin
            Result := RetryPolicy.Execute<TOrderDto>(
              function: TOrderDto
              begin
                Result := OrderRepository.GetById(AOrderId);
              end
            );
          end
        );
      end,
      // Default: Empty order
      nil
    );

    // Fallback: Get from cache
    Fallback.WithFallback(
      function: TOrderDto
      begin
        var Cached := Cache.Get('order:' + IntToStr(AOrderId));
        if Cached <> '' then
          Result := JsonToOrderDto(Cached)
        else
          raise ENotFoundException.Create('Not in cache');
      end
    );

    Result := Fallback.Execute;

    if Assigned(Result) then
      WriteLn('Order found: ', Result.OrderNumber)
    else
      WriteLn('Order not found (graceful degradation)');

  finally
    RetryPolicy.Free;
    Fallback.Free;
  end;
end;

begin
  TAppConfiguration.Initialize;

  SetupHighAvailability;
  ProcessOrderWithHA(1001);

  ReadLn;
end.
```

## 6. สรุป High Availability Patterns

| Pattern | จุดประสงค์ | เมื่อใช้ |
|---------|----------|---------|
| Circuit Breaker | ป้องกัน Cascade Failure | External dependencies |
| Retry + Backoff | จัดการ Transient Errors | Network/DB calls |
| Fallback | Graceful Degradation | Critical features |
| Bulkhead | แยก Resource pools | High-traffic systems |
| Health Check | Monitor system health | All production systems |
| Feature Flags | Safe deployment | Risky features |

**HA Design Principles:**
1. **Assume failure** - ออกแบบให้รับมือกับ Failure
2. **Fail fast** - ตรวจสอบ Error เร็วและ Fail อย่าง Graceful
3. **Isolate failures** - ป้องกัน Cascade Failure
4. **Degrade gracefully** - ลด Feature บางส่วนดีกว่า Crash ทั้งหมด
5. **Monitor everything** - ไม่รู้ = ไม่สามารถแก้ไข
