# ตอนที่ 88: Application Monitoring กับ Pascal/Lazarus

## บทนำ: การ Monitor ระบบ Production

การ Monitor คือหัวใจสำคัญของระบบ Production ที่ต้องทำงานตลอด 24 ชั่วโมง

## 1. Metrics Collection System

```pascal
// uMetrics.pas - Metrics Collection
unit uMetrics;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections;

type
  TMetricType = (mtCounter, mtGauge, mtHistogram, mtSummary);

  TMetric = class abstract
  protected
    FName: string;
    FHelp: string;
    FLabels: TStringList;
    FCriticalSection: TCriticalSection;
  public
    constructor Create(const AName, AHelp: string);
    destructor Destroy; override;
    function GetPrometheusLine: string; virtual; abstract;
    property Name: string read FName;
  end;

  // Counter - Only goes up (requests, errors)
  TCounter = class(TMetric)
  private
    FValue: Double;
    FLabelValues: TDictionary<string, Double>;
  public
    constructor Create(const AName, AHelp: string);
    destructor Destroy; override;
    procedure Inc_(const ALabels: array of string; AAmount: Double = 1.0);
    function GetValue(const ALabels: array of string = []): Double;
    function GetPrometheusLine: string; override;
  end;

  // Gauge - Can go up and down (memory usage, active connections)
  TGauge = class(TMetric)
  private
    FValue: Double;
    FLabelValues: TDictionary<string, Double>;
  public
    constructor Create(const AName, AHelp: string);
    destructor Destroy; override;
    procedure Set_(AValue: Double; const ALabels: array of string = []);
    procedure Inc_(const ALabels: array of string; AAmount: Double = 1.0);
    procedure Dec_(const ALabels: array of string; AAmount: Double = 1.0);
    function GetPrometheusLine: string; override;
  end;

  // Histogram - Distribution of values (response times)
  THistogram = class(TMetric)
  private
    FBuckets: TArray<Double>;
    FBucketCounts: TArray<Int64>;
    FSumValue: Double;
    FCountValue: Int64;
    FCriticalSection2: TCriticalSection;
  public
    constructor Create(const AName, AHelp: string;
      const ABuckets: TArray<Double> = nil);
    destructor Destroy; override;
    procedure Observe(AValue: Double);
    function GetPrometheusLine: string; override;
  end;

  // Metrics Registry
  TMetricsRegistry = class
  private
    FMetrics: TObjectDictionary<string, TMetric>;
    FCriticalSection: TCriticalSection;
    class var FInstance: TMetricsRegistry;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Register(const AMetric: TMetric);
    function GetMetric(const AName: string): TMetric;
    function GetAllMetrics: TObjectDictionary<string, TMetric>;
    function GetPrometheusOutput: string;
    class function Instance: TMetricsRegistry;
    class constructor ClassCreate;
    class destructor ClassDestroy;
  end;

  // Timer Helper for Histograms
  TMetricsTimer = class
  private
    FHistogram: THistogram;
    FLabels: TArray<string>;
    FStartTime: TDateTime;
  public
    constructor Create(const AHistogram: THistogram;
      const ALabels: TArray<string> = nil);
    destructor Destroy; override; // Records duration on destroy
  end;

  // Application Metrics Collector
  TAppMetrics = class
  public
    // HTTP Metrics
    HttpRequestsTotal: TCounter;
    HttpRequestDuration: THistogram;
    HttpActiveRequests: TGauge;

    // Database Metrics
    DbQueryDuration: THistogram;
    DbConnectionsActive: TGauge;
    DbErrorsTotal: TCounter;

    // Business Metrics
    OrdersCreated: TCounter;
    OrdersTotal: TGauge;
    RevenueTotal: TCounter;

    // System Metrics
    MemoryUsageBytes: TGauge;
    GoroutinesCount: TGauge;

    constructor Create;
    destructor Destroy; override;
    procedure CollectSystemMetrics;
  end;

implementation

constructor TMetric.Create(const AName, AHelp: string);
begin
  inherited Create;
  FName := AName;
  FHelp := AHelp;
  FCriticalSection := TCriticalSection.Create;
end;

destructor TMetric.Destroy;
begin
  FCriticalSection.Free;
  inherited Destroy;
end;

// Counter
constructor TCounter.Create(const AName, AHelp: string);
begin
  inherited Create(AName, AHelp);
  FValue := 0;
  FLabelValues := TDictionary<string, Double>.Create;
end;

destructor TCounter.Destroy;
begin
  FLabelValues.Free;
  inherited Destroy;
end;

procedure TCounter.Inc_(const ALabels: array of string; AAmount: Double);
var
  Key: string;
  CurrentValue: Double;
begin
  FCriticalSection.Acquire;
  try
    Key := String.Join(',', ALabels);
    if Key = '' then
    begin
      FValue := FValue + AAmount;
    end
    else
    begin
      if FLabelValues.TryGetValue(Key, CurrentValue) then
        FLabelValues[Key] := CurrentValue + AAmount
      else
        FLabelValues.Add(Key, AAmount);
    end;
  finally
    FCriticalSection.Release;
  end;
end;

function TCounter.GetPrometheusLine: string;
var
  Lines: TStringList;
  KV: TPair<string, Double>;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('# HELP ' + FName + ' ' + FHelp);
    Lines.Add('# TYPE ' + FName + ' counter');

    FCriticalSection.Acquire;
    try
      if FValue > 0 then
        Lines.Add(FName + ' ' + FloatToStr(FValue));

      for KV in FLabelValues do
        Lines.Add(Format('%s{%s} %s', [FName, KV.Key, FloatToStr(KV.Value)]));
    finally
      FCriticalSection.Release;
    end;

    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

// Histogram
constructor THistogram.Create(const AName, AHelp: string;
  const ABuckets: TArray<Double>);
var
  DefaultBuckets: TArray<Double>;
begin
  inherited Create(AName, AHelp);
  FCriticalSection2 := TCriticalSection.Create;

  if Assigned(ABuckets) and (Length(ABuckets) > 0) then
    FBuckets := ABuckets
  else
  begin
    // Default HTTP response time buckets (ms)
    DefaultBuckets := [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000, 10000];
    FBuckets := DefaultBuckets;
  end;

  SetLength(FBucketCounts, Length(FBuckets) + 1); // +1 for +Inf
  FSumValue := 0;
  FCountValue := 0;
end;

destructor THistogram.Destroy;
begin
  FCriticalSection2.Free;
  inherited Destroy;
end;

procedure THistogram.Observe(AValue: Double);
var
  I: Integer;
begin
  FCriticalSection2.Acquire;
  try
    FSumValue := FSumValue + AValue;
    Inc(FCountValue);

    for I := 0 to High(FBuckets) do
      if AValue <= FBuckets[I] then
        Inc(FBucketCounts[I]);

    // +Inf bucket always counts
    Inc(FBucketCounts[High(FBucketCounts)]);
  finally
    FCriticalSection2.Release;
  end;
end;

function THistogram.GetPrometheusLine: string;
var
  Lines: TStringList;
  I: Integer;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('# HELP ' + FName + ' ' + FHelp);
    Lines.Add('# TYPE ' + FName + ' histogram');

    FCriticalSection2.Acquire;
    try
      for I := 0 to High(FBuckets) do
        Lines.Add(Format('%s_bucket{le="%g"} %d',
          [FName, FBuckets[I], FBucketCounts[I]]));

      Lines.Add(Format('%s_bucket{le="+Inf"} %d',
        [FName, FBucketCounts[High(FBucketCounts)]]));
      Lines.Add(Format('%s_sum %g', [FName, FSumValue]));
      Lines.Add(Format('%s_count %d', [FName, FCountValue]));
    finally
      FCriticalSection2.Release;
    end;

    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

// Metrics Registry
function TMetricsRegistry.GetPrometheusOutput: string;
var
  Lines: TStringList;
  Metric: TMetric;
begin
  Lines := TStringList.Create;
  try
    FCriticalSection.Acquire;
    try
      for Metric in FMetrics.Values do
        Lines.Add(Metric.GetPrometheusLine);
    finally
      FCriticalSection.Release;
    end;
    Result := Lines.Text;
  finally
    Lines.Free;
  end;
end;

// Timer
constructor TMetricsTimer.Create(const AHistogram: THistogram;
  const ALabels: TArray<string>);
begin
  inherited Create;
  FHistogram := AHistogram;
  FLabels := ALabels;
  FStartTime := Now;
end;

destructor TMetricsTimer.Destroy;
var
  ElapsedMs: Double;
begin
  ElapsedMs := (Now - FStartTime) * MSecsPerDay;
  if Assigned(FHistogram) then
    FHistogram.Observe(ElapsedMs);
  inherited Destroy;
end;

// Application Metrics
constructor TAppMetrics.Create;
begin
  inherited Create;
  var R := TMetricsRegistry.Instance;

  // HTTP Metrics
  HttpRequestsTotal := TCounter.Create(
    'http_requests_total', 'Total HTTP requests');
  R.Register(HttpRequestsTotal);

  HttpRequestDuration := THistogram.Create(
    'http_request_duration_ms', 'HTTP request duration in milliseconds',
    [10, 50, 100, 200, 500, 1000, 2000, 5000]);
  R.Register(HttpRequestDuration);

  HttpActiveRequests := TGauge.Create(
    'http_active_requests', 'Active HTTP requests');
  R.Register(HttpActiveRequests);

  // DB Metrics
  DbQueryDuration := THistogram.Create(
    'db_query_duration_ms', 'Database query duration');
  R.Register(DbQueryDuration);

  DbConnectionsActive := TGauge.Create(
    'db_connections_active', 'Active database connections');
  R.Register(DbConnectionsActive);

  // Business Metrics
  OrdersCreated := TCounter.Create(
    'orders_created_total', 'Total orders created');
  R.Register(OrdersCreated);

  RevenueTotal := TCounter.Create(
    'revenue_total_thb', 'Total revenue in THB');
  R.Register(RevenueTotal);

  // System Metrics
  MemoryUsageBytes := TGauge.Create(
    'process_memory_bytes', 'Process memory usage');
  R.Register(MemoryUsageBytes);
end;

procedure TAppMetrics.CollectSystemMetrics;
begin
  // Collect memory usage
  {$IFDEF LINUX}
  var MemInfo := GetMemInfo;
  MemoryUsageBytes.Set_(MemInfo.RSS);
  {$ENDIF}
end;

end.
```

## 2. Prometheus Metrics Endpoint

```pascal
// uPrometheusEndpoint.pas
unit uPrometheusEndpoint;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpserver, httpdefs,
  uMetrics, uBaseMicroservice;

type
  TPrometheusMiddleware = class
  private
    FMetrics: TAppMetrics;

  public
    constructor Create(const AMetrics: TAppMetrics);

    procedure BeforeRequest(ARequest: TFPHTTPConnectionRequest);
    procedure AfterRequest(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse;
      AStartTime: TDateTime);
  end;

  // Metrics HTTP Handler (for Prometheus scraping)
  TMetricsHandler = class
  public
    class procedure Handle(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
  end;

implementation

constructor TPrometheusMiddleware.Create(const AMetrics: TAppMetrics);
begin
  inherited Create;
  FMetrics := AMetrics;
end;

procedure TPrometheusMiddleware.BeforeRequest(ARequest: TFPHTTPConnectionRequest);
begin
  FMetrics.HttpActiveRequests.Inc_([], 1);
end;

procedure TPrometheusMiddleware.AfterRequest(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse; AStartTime: TDateTime);
var
  DurationMs: Double;
  Status: string;
begin
  DurationMs := (Now - AStartTime) * MSecsPerDay;
  Status := IntToStr(AResponse.Code);

  FMetrics.HttpActiveRequests.Dec_([], 1);
  FMetrics.HttpRequestsTotal.Inc_(
    [ARequest.Method, ARequest.PathInfo, Status]);
  FMetrics.HttpRequestDuration.Observe(DurationMs);
end;

class procedure TMetricsHandler.Handle(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
begin
  AResponse.Code := 200;
  AResponse.ContentType := 'text/plain; version=0.0.4; charset=utf-8';
  AResponse.Content := TMetricsRegistry.Instance.GetPrometheusOutput;
end;

end.
```

## 3. Structured Logging with JSON

```pascal
// uStructuredLogger.pas - JSON Structured Logging
unit uStructuredLogger;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, SyncObjs, Generics.Collections;

type
  TLogLevel = (llTrace, llDebug, llInfo, llWarning, llError, llCritical);

  TLogEntry = class
  public
    Timestamp: TDateTime;
    Level: TLogLevel;
    Message: string;
    Service: string;
    CorrelationId: string;
    TraceId: string;
    SpanId: string;
    Fields: TStringList;
    Exception_: string;
    StackTrace: string;

    constructor Create;
    destructor Destroy; override;
    function ToJSON: string;
  end;

  ILogSink = interface
    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

  // JSON Console Sink (for structured logging)
  TJsonConsoleSink = class(TInterfacedObject, ILogSink)
  public
    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

  // Loki-compatible Log Sink
  TLokiLogSink = class(TInterfacedObject, ILogSink)
  private
    FLokiUrl: string;
    FJobName: string;
    FBuffer: TList<TLogEntry>;
    FCriticalSection: TCriticalSection;
    FFlushTimer: TTimer;
    FHttpClient: TFPHTTPClient;

    procedure FlushBuffer;

  public
    constructor Create(const ALokiUrl, AJobName: string);
    destructor Destroy; override;

    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

  // Contextual Logger (carries context fields)
  TContextLogger = class(TInterfacedObject, ILogger)
  private
    FBaseLogger: ILogger;
    FContext: TStringList;
    FService: string;
    FCorrelationId: string;

  public
    constructor Create(const ABaseLogger: ILogger; const AService: string);
    destructor Destroy; override;

    function WithField(const AKey, AValue: string): TContextLogger;
    function WithCorrelationId(const AId: string): TContextLogger;
    function WithTraceId(const ATraceId, ASpanId: string): TContextLogger;
    function WithRequest(ARequest: TFPHTTPConnectionRequest): TContextLogger;

    procedure Trace(const AMessage: string);
    procedure Debug(const AMessage: string);
    procedure Info(const AMessage: string);
    procedure Warning(const AMessage: string);
    procedure Error(const AMessage: string; const AException: Exception = nil);
    procedure Critical(const AMessage: string; const AException: Exception = nil);
  end;

implementation

function TLogEntry.ToJSON: string;
const
  LevelNames: array[TLogLevel] of string =
    ('trace', 'debug', 'info', 'warning', 'error', 'critical');
var
  Json: TJSONObject;
  FieldsObj: TJSONObject;
  I: Integer;
begin
  Json := TJSONObject.Create;
  try
    Json.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss.zzz"Z"', Timestamp));
    Json.Add('level', LevelNames[Level]);
    Json.Add('message', Message);
    Json.Add('service', Service);

    if CorrelationId <> '' then
      Json.Add('correlationId', CorrelationId);
    if TraceId <> '' then
      Json.Add('traceId', TraceId);
    if SpanId <> '' then
      Json.Add('spanId', SpanId);

    if Assigned(Fields) and (Fields.Count > 0) then
    begin
      FieldsObj := TJSONObject.Create;
      for I := 0 to Fields.Count - 1 do
        FieldsObj.Add(Fields.Names[I], Fields.ValueFromIndex[I]);
      Json.Add('fields', FieldsObj);
    end;

    if Exception_ <> '' then
    begin
      var ExcObj := TJSONObject.Create;
      ExcObj.Add('message', Exception_);
      if StackTrace <> '' then
        ExcObj.Add('stackTrace', StackTrace);
      Json.Add('exception', ExcObj);
    end;

    Result := Json.AsJSON;
  finally
    Json.Free;
  end;
end;

procedure TJsonConsoleSink.Write(const AEntry: TLogEntry);
begin
  WriteLn(AEntry.ToJSON);
end;

// Loki Sink (Grafana Loki)
procedure TLokiLogSink.Write(const AEntry: TLogEntry);
begin
  FCriticalSection.Acquire;
  try
    FBuffer.Add(AEntry);
    if FBuffer.Count >= 100 then
      FlushBuffer;
  finally
    FCriticalSection.Release;
  end;
end;

procedure TLokiLogSink.FlushBuffer;
var
  StreamsObj: TJSONObject;
  StreamArray: TJSONArray;
  StreamObj: TJSONObject;
  ValuesArray: TJSONArray;
  EntryArray: TJSONArray;
  Entry: TLogEntry;
  Body: string;
const
  LevelNames: array[TLogLevel] of string =
    ('trace', 'debug', 'info', 'warning', 'error', 'critical');
begin
  if FBuffer.Count = 0 then Exit;

  try
    StreamsObj := TJSONObject.Create;
    StreamArray := TJSONArray.Create;

    StreamObj := TJSONObject.Create;
    var Labels := TJSONObject.Create;
    Labels.Add('job', FJobName);
    Labels.Add('service', 'api');
    StreamObj.Add('stream', Labels);

    ValuesArray := TJSONArray.Create;
    for Entry in FBuffer do
    begin
      EntryArray := TJSONArray.Create;
      // Loki timestamp in nanoseconds
      EntryArray.Add(IntToStr(
        Round(Entry.Timestamp * SecsPerDay * 1000000000)));
      EntryArray.Add(Entry.ToJSON);
      ValuesArray.Add(EntryArray);
    end;

    StreamObj.Add('values', ValuesArray);
    StreamArray.Add(StreamObj);
    StreamsObj.Add('streams', StreamArray);

    Body := StreamsObj.AsJSON;
    StreamsObj.Free;

    var ResponseStream := TStringStream.Create;
    try
      FHttpClient.AddHeader('Content-Type', 'application/json');
      FHttpClient.RequestBody := TStringStream.Create(Body, CP_UTF8);
      FHttpClient.Post(FLokiUrl + '/loki/api/v1/push', ResponseStream);
    finally
      ResponseStream.Free;
    end;

    FBuffer.Clear;
  except
    on E: Exception do
      WriteLn('Failed to flush to Loki: ', E.Message);
  end;
end;

end.
```

## 4. APM - Application Performance Monitoring

```pascal
// uTracing.pas - Distributed Tracing (OpenTelemetry-compatible)
unit uTracing;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, Generics.Collections, fpjson;

type
  TSpanStatus = (ssOk, ssError, ssUnset);

  TSpan = class
  private
    FTraceId: string;
    FSpanId: string;
    FParentSpanId: string;
    FOperationName: string;
    FService: string;
    FStartTime: TDateTime;
    FEndTime: TDateTime;
    FStatus: TSpanStatus;
    FAttributes: TStringList;
    FEvents: TList<TJSONObject>;
    FFinished: Boolean;

  public
    constructor Create(const ATraceId, ASpanId, AParentSpanId,
      AOperationName, AService: string);
    destructor Destroy; override;

    procedure SetAttribute(const AKey, AValue: string);
    procedure SetAttributeInt(const AKey: string; AValue: Int64);
    procedure SetAttributeFloat(const AKey: string; AValue: Double);
    procedure AddEvent(const AName: string;
      const AAttributes: TStringList = nil);
    procedure SetStatus(AStatus: TSpanStatus; const AMessage: string = '');
    procedure Finish;

    function ToJson: TJSONObject;

    property TraceId: string read FTraceId;
    property SpanId: string read FSpanId;
    property OperationName: string read FOperationName;
    property StartTime: TDateTime read FStartTime;
    property IsFinished: Boolean read FFinished;
  end;

  ITracer = interface
    function StartSpan(const AOperationName: string): TSpan;
    function StartChildSpan(const AParentSpan: TSpan;
      const AOperationName: string): TSpan;
    function StartFromContext(const ATraceId, AParentSpanId: string;
      const AOperationName: string): TSpan;
    procedure FinishSpan(const ASpan: TSpan);
    function GetCurrentSpan: TSpan;
  end;

  TJaegerTracer = class(TInterfacedObject, ITracer)
  private
    FServiceName: string;
    FJaegerUrl: string;
    FActiveSpans: TStack<TSpan>;
    FCriticalSection: TCriticalSection;
    FBatchBuffer: TObjectList<TSpan>;
    FBatchSize: Integer;

    procedure SendBatch;
    function GenerateId(ALength: Integer = 16): string;

  public
    constructor Create(const AServiceName: string;
      const AJaegerUrl: string = 'http://localhost:14268');
    destructor Destroy; override;

    function StartSpan(const AOperationName: string): TSpan;
    function StartChildSpan(const AParentSpan: TSpan;
      const AOperationName: string): TSpan;
    function StartFromContext(const ATraceId, AParentSpanId: string;
      const AOperationName: string): TSpan;
    procedure FinishSpan(const ASpan: TSpan);
    function GetCurrentSpan: TSpan;
  end;

  // Span wrapper for RAII pattern
  TSpanScope = class
  private
    FTracer: ITracer;
    FSpan: TSpan;
  public
    constructor Create(const ATracer: ITracer; const AOperationName: string;
      const AParentSpan: TSpan = nil);
    destructor Destroy; override; // Finishes span on destroy
    property Span: TSpan read FSpan;
  end;

implementation

constructor TSpan.Create(const ATraceId, ASpanId, AParentSpanId,
  AOperationName, AService: string);
begin
  inherited Create;
  FTraceId := ATraceId;
  FSpanId := ASpanId;
  FParentSpanId := AParentSpanId;
  FOperationName := AOperationName;
  FService := AService;
  FStartTime := Now;
  FStatus := ssUnset;
  FAttributes := TStringList.Create;
  FEvents := TList<TJSONObject>.Create;
  FFinished := False;
end;

destructor TSpan.Destroy;
var
  Event: TJSONObject;
begin
  if not FFinished then Finish;
  for Event in FEvents do Event.Free;
  FEvents.Free;
  FAttributes.Free;
  inherited Destroy;
end;

procedure TSpan.Finish;
begin
  if not FFinished then
  begin
    FEndTime := Now;
    FFinished := True;
  end;
end;

procedure TSpan.SetAttribute(const AKey, AValue: string);
begin
  FAttributes.Values[AKey] := AValue;
end;

procedure TSpan.AddEvent(const AName: string; const AAttributes: TStringList);
var
  Event: TJSONObject;
  I: Integer;
begin
  Event := TJSONObject.Create;
  Event.Add('name', AName);
  Event.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss.zzz"Z"', Now));

  if Assigned(AAttributes) then
  begin
    var Attrs := TJSONObject.Create;
    for I := 0 to AAttributes.Count - 1 do
      Attrs.Add(AAttributes.Names[I], AAttributes.ValueFromIndex[I]);
    Event.Add('attributes', Attrs);
  end;

  FEvents.Add(Event);
end;

function TSpan.ToJson: TJSONObject;
var
  I: Integer;
  AttrsObj: TJSONObject;
  EventsArray: TJSONArray;
begin
  Result := TJSONObject.Create;
  Result.Add('traceId', FTraceId);
  Result.Add('spanId', FSpanId);
  if FParentSpanId <> '' then
    Result.Add('parentSpanId', FParentSpanId);
  Result.Add('operationName', FOperationName);
  Result.Add('service', FService);
  Result.Add('startTime', Round(FStartTime * MSecsPerDay));
  if FFinished then
    Result.Add('duration',
      Round((FEndTime - FStartTime) * MSecsPerDay));

  case FStatus of
    ssOk: Result.Add('status', 'ok');
    ssError: Result.Add('status', 'error');
    ssUnset: Result.Add('status', 'unset');
  end;

  if FAttributes.Count > 0 then
  begin
    AttrsObj := TJSONObject.Create;
    for I := 0 to FAttributes.Count - 1 do
      AttrsObj.Add(FAttributes.Names[I], FAttributes.ValueFromIndex[I]);
    Result.Add('attributes', AttrsObj);
  end;

  if FEvents.Count > 0 then
  begin
    EventsArray := TJSONArray.Create;
    for var Event in FEvents do
      EventsArray.Add(Event.Clone);
    Result.Add('events', EventsArray);
  end;
end;

// Span Scope
constructor TSpanScope.Create(const ATracer: ITracer;
  const AOperationName: string; const AParentSpan: TSpan);
begin
  inherited Create;
  FTracer := ATracer;
  if Assigned(AParentSpan) then
    FSpan := FTracer.StartChildSpan(AParentSpan, AOperationName)
  else
    FSpan := FTracer.StartSpan(AOperationName);
end;

destructor TSpanScope.Destroy;
begin
  if Assigned(FSpan) and not FSpan.IsFinished then
    FTracer.FinishSpan(FSpan);
  inherited Destroy;
end;

end.
```

## 5. Alerting System

```pascal
// uAlerting.pas - Alert Management
unit uAlerting;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, fpjson, fphttpclient;

type
  TAlertSeverity = (asInfo, asWarning, asCritical);

  TAlert = class
  public
    Name: string;
    Severity: TAlertSeverity;
    Message: string;
    Service: string;
    Labels: TStringList;
    FiredAt: TDateTime;
    ResolvedAt: TDateTime;
    IsResolved: Boolean;

    constructor Create;
    destructor Destroy; override;
  end;

  IAlertChannel = interface
    procedure SendAlert(const AAlert: TAlert);
    procedure ResolveAlert(const AAlert: TAlert);
  end;

  // Slack Alert Channel
  TSlackAlertChannel = class(TInterfacedObject, IAlertChannel)
  private
    FWebhookUrl: string;
    FChannel: string;

  public
    constructor Create(const AWebhookUrl, AChannel: string);

    procedure SendAlert(const AAlert: TAlert);
    procedure ResolveAlert(const AAlert: TAlert);
  end;

  // Email Alert Channel
  TEmailAlertChannel = class(TInterfacedObject, IAlertChannel)
  private
    FSmtpHost: string;
    FFromAddress: string;
    FToAddresses: TStringList;

  public
    constructor Create(const ASmtpHost, AFromAddress: string;
      const AToAddresses: TArray<string>);
    destructor Destroy; override;

    procedure SendAlert(const AAlert: TAlert);
    procedure ResolveAlert(const AAlert: TAlert);
  end;

  // PagerDuty Integration
  TPagerDutyChannel = class(TInterfacedObject, IAlertChannel)
  private
    FApiKey: string;
    FServiceId: string;

  public
    constructor Create(const AApiKey, AServiceId: string);

    procedure SendAlert(const AAlert: TAlert);
    procedure ResolveAlert(const AAlert: TAlert);
  end;

  // Alert Rule
  TAlertRule = class
  public
    Name: string;
    Condition: TFunc<Boolean>;
    Severity: TAlertSeverity;
    Message: TFunc<string>;
    CooldownSeconds: Integer;
    LastFired: TDateTime;
    IsFiring: Boolean;
  end;

  // Alert Manager
  TAlertManager = class
  private
    FRules: TObjectList<TAlertRule>;
    FChannels: TInterfaceList;
    FActiveAlerts: TObjectDictionary<string, TAlert>;
    FCriticalSection: TCriticalSection;
    FEvaluationTimer: TTimer;

    procedure EvaluateRules;
    procedure FireAlert(const ARule: TAlertRule);
    procedure ResolveAlert(const ARule: TAlertRule);

  public
    constructor Create;
    destructor Destroy; override;

    procedure AddRule(const ARule: TAlertRule);
    procedure AddChannel(const AChannel: IAlertChannel);
    procedure Start(AIntervalSeconds: Integer = 30);
    procedure Stop;
  end;

implementation

// Slack Alert
procedure TSlackAlertChannel.SendAlert(const AAlert: TAlert);
const
  SeverityColors: array[TAlertSeverity] of string =
    ('#439FE0', '#FFAA00', '#FF0000');  // Blue, Yellow, Red
  SeverityEmoji: array[TAlertSeverity] of string =
    (':information_source:', ':warning:', ':fire:');
var
  Payload: TJSONObject;
  Attachment: TJSONObject;
  Fields: TJSONArray;
  Field: TJSONObject;
  Client: TFPHTTPClient;
  Response: TStringStream;
begin
  Payload := TJSONObject.Create;
  try
    Payload.Add('channel', FChannel);
    Payload.Add('username', 'AlertBot');

    Attachment := TJSONObject.Create;
    Attachment.Add('color', SeverityColors[AAlert.Severity]);
    Attachment.Add('title', Format('%s %s: %s',
      [SeverityEmoji[AAlert.Severity], AAlert.Service, AAlert.Name]));
    Attachment.Add('text', AAlert.Message);
    Attachment.Add('ts', Round(AAlert.FiredAt * SecsPerDay));

    Fields := TJSONArray.Create;
    Field := TJSONObject.Create;
    Field.Add('title', 'Service');
    Field.Add('value', AAlert.Service);
    Field.Add('short', True);
    Fields.Add(Field);

    Field := TJSONObject.Create;
    Field.Add('title', 'Severity');
    Field.Add('value', ['info', 'warning', 'critical'][Ord(AAlert.Severity)]);
    Field.Add('short', True);
    Fields.Add(Field);

    Attachment.Add('fields', Fields);

    var Attachments := TJSONArray.Create;
    Attachments.Add(Attachment);
    Payload.Add('attachments', Attachments);

    Client := TFPHTTPClient.Create(nil);
    Response := TStringStream.Create;
    try
      Client.AddHeader('Content-Type', 'application/json');
      Client.RequestBody := TStringStream.Create(Payload.AsJSON, CP_UTF8);
      Client.Post(FWebhookUrl, Response);
    finally
      Response.Free;
      Client.Free;
    end;
  finally
    Payload.Free;
  end;
end;

// Alert Manager
procedure TAlertManager.EvaluateRules;
var
  Rule: TAlertRule;
  ShouldFire: Boolean;
begin
  FCriticalSection.Acquire;
  try
    for Rule in FRules do
    begin
      try
        ShouldFire := Rule.Condition();

        if ShouldFire and not Rule.IsFiring then
        begin
          // Check cooldown
          if (Rule.LastFired = 0) or
             ((Now - Rule.LastFired) * SecsPerDay > Rule.CooldownSeconds) then
          begin
            FireAlert(Rule);
            Rule.IsFiring := True;
            Rule.LastFired := Now;
          end;
        end
        else if not ShouldFire and Rule.IsFiring then
        begin
          ResolveAlert(Rule);
          Rule.IsFiring := False;
        end;
      except
        on E: Exception do
          // Log but continue with other rules
      end;
    end;
  finally
    FCriticalSection.Release;
  end;
end;

procedure TAlertManager.FireAlert(const ARule: TAlertRule);
var
  Alert: TAlert;
  Channel: IAlertChannel;
  I: Integer;
begin
  Alert := TAlert.Create;
  Alert.Name := ARule.Name;
  Alert.Severity := ARule.Severity;
  Alert.Message := ARule.Message();
  Alert.FiredAt := Now;
  Alert.IsResolved := False;

  FActiveAlerts.AddOrSetValue(ARule.Name, Alert);

  for I := 0 to FChannels.Count - 1 do
  begin
    Channel := IAlertChannel(FChannels[I]);
    try
      Channel.SendAlert(Alert);
    except
      // Don't let channel errors prevent other channels
    end;
  end;
end;

end.
```

## 6. Grafana Dashboard Configuration

```pascal
// uDashboard.pas - Dashboard Metrics for Grafana
unit uDashboard;

{$mode objfpc}{$H+}

interface

// Grafana Dashboard JSON Configuration (เพื่อ Reference)
// สร้าง Dashboard ด้วย API หรือ JSON provisioning

const
  // Prometheus Queries สำหรับ Dashboard
  QUERY_HTTP_REQUESTS_RATE =
    'rate(http_requests_total{service="$service"}[5m])';

  QUERY_HTTP_ERROR_RATE =
    'sum(rate(http_requests_total{status=~"5.."}[5m])) / ' +
    'sum(rate(http_requests_total[5m])) * 100';

  QUERY_HTTP_P95_LATENCY =
    'histogram_quantile(0.95, ' +
    '  rate(http_request_duration_ms_bucket[5m]))';

  QUERY_DB_ACTIVE_CONNECTIONS =
    'db_connections_active{service="$service"}';

  QUERY_MEMORY_USAGE =
    'process_memory_bytes{service="$service"} / 1024 / 1024';

  QUERY_ORDERS_PER_MINUTE =
    'rate(orders_created_total[1m]) * 60';

  QUERY_REVENUE_PER_HOUR =
    'rate(revenue_total_thb[1h]) * 3600';

type
  // Metric Reporter สำหรับ Business Dashboard
  TBusinessMetricsReporter = class
  private
    FMetrics: TAppMetrics;
    FDatabase: TSQLConnection;

    procedure CollectOrderMetrics;
    procedure CollectRevenueMetrics;
    procedure CollectCustomerMetrics;

  public
    constructor Create(const AMetrics: TAppMetrics;
      const ADatabase: TSQLConnection);
    procedure Collect;
  end;

implementation

procedure TBusinessMetricsReporter.Collect;
begin
  FMetrics.CollectSystemMetrics;
  CollectOrderMetrics;
  CollectRevenueMetrics;
  CollectCustomerMetrics;
end;

procedure TBusinessMetricsReporter.CollectOrderMetrics;
var
  Query: TSQLQuery;
begin
  Query := TSQLQuery.Create(nil);
  try
    Query.DataBase := FDatabase;
    Query.SQL.Text :=
      'SELECT status, COUNT(*) as cnt ' +
      'FROM orders ' +
      'WHERE created_at >= NOW() - INTERVAL ''1 hour'' ' +
      'GROUP BY status';
    Query.Open;

    while not Query.EOF do
    begin
      FMetrics.OrdersTotal.Set_(
        Query.FieldByName('cnt').AsFloat,
        [Query.FieldByName('status').AsString]
      );
      Query.Next;
    end;
  finally
    Query.Free;
  end;
end;

end.
```

## 7. สรุป Monitoring Stack

**Monitoring Stack ที่แนะนำ:**
```
Pascal App
    │
    ├── Metrics → Prometheus → Grafana (Dashboards)
    │
    ├── Logs → Loki → Grafana (Log Explorer)
    │
    ├── Traces → Jaeger/Tempo → Grafana (Distributed Tracing)
    │
    └── Alerts → AlertManager → Slack/PagerDuty/Email
```

**สิ่งที่ต้อง Monitor:**
1. **Infrastructure**: CPU, Memory, Disk, Network
2. **Application**: Request rate, Error rate, Latency (RED)
3. **Business**: Orders, Revenue, Active Users
4. **Dependencies**: DB, Cache, External APIs

**Golden Signals (Google SRE):**
- **Latency** - ความเร็วในการตอบสนอง
- **Traffic** - ปริมาณ Request
- **Errors** - อัตรา Error
- **Saturation** - การใช้ Resource

**SLI/SLO/SLA:**
- SLI (Service Level Indicator): การวัดจริง เช่น 99.5% requests < 200ms
- SLO (Service Level Objective): เป้าหมาย เช่น 99% availability
- SLA (Service Level Agreement): สัญญากับลูกค้า
