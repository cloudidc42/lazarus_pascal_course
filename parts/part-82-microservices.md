# ตอนที่ 82: Microservices Architecture กับ Pascal/Lazarus

## บทนำ: สถาปัตยกรรม Microservices

Microservices คือรูปแบบสถาปัตยกรรมที่แบ่งแอปพลิเคชันออกเป็นบริการขนาดเล็กหลายตัว แต่ละบริการทำงานอิสระ มี Database ของตัวเอง และสื่อสารกันผ่าน API หรือ Message Queue

## 1. หลักการ Microservices

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway                          │
│         (Authentication, Routing, Rate Limiting)        │
└────────┬──────────┬──────────┬──────────┬──────────────┘
         │          │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
    │Customer│ │ Order  │ │Product │ │Payment │
    │Service │ │Service │ │Service │ │Service │
    └────┬───┘ └────┬───┘ └───┬────┘ └──┬─────┘
         │          │          │          │
    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
    │CustomerDB││OrderDB │ │ProductDB│ │PaymentDB│
    └────────┘ └────────┘ └────────┘ └────────┘
```

## 2. Building Microservices with fphttpserver

### Base Microservice Class

```pascal
// uBaseMicroservice.pas
unit uBaseMicroservice;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpserver, httpdefs, fpjson, jsonparser,
  uLogger, uConfiguration, uHealthCheck;

type
  THttpMethod = (hmGet, hmPost, hmPut, hmPatch, hmDelete);

  TRouteHandler = procedure(ARequest: TFPHTTPConnectionRequest;
    AResponse: TFPHTTPConnectionResponse) of object;

  TRoute = record
    Method: THttpMethod;
    Path: string;
    Handler: TRouteHandler;
    RequiresAuth: Boolean;
  end;

  TBaseMicroservice = class
  private
    FServer: TFPHTTPServer;
    FRoutes: array of TRoute;
    FLogger: ILogger;
    FConfig: TAppConfiguration;
    FServiceName: string;
    FPort: Integer;

    procedure OnRequest(Sender: TObject;
      var ARequest: TFPHTTPConnectionRequest;
      var AResponse: TFPHTTPConnectionResponse);
    function FindRoute(const AMethod: THttpMethod;
      const APath: string): Integer;
    function ParseMethod(const AMethodStr: string): THttpMethod;
    procedure HandleCors(AResponse: TFPHTTPConnectionResponse);
    function ValidateToken(const AToken: string): Boolean;
    procedure SetupDefaultRoutes;

  protected
    procedure RegisterRoute(const AMethod: THttpMethod; const APath: string;
      const AHandler: TRouteHandler; ARequiresAuth: Boolean = True);

    // Default Route Handlers
    procedure HandleHealth(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleMetrics(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleReady(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);

    // Helper Methods
    procedure WriteJson(AResponse: TFPHTTPConnectionResponse;
      const AJson: string; AStatusCode: Integer = 200);
    procedure WriteError(AResponse: TFPHTTPConnectionResponse;
      const AMessage: string; AStatusCode: Integer = 500);
    function ReadJsonBody(ARequest: TFPHTTPConnectionRequest): TJSONObject;
    function GetPathParam(const APath, APattern: string;
      const AParamName: string): string;
    function GetQueryParam(ARequest: TFPHTTPConnectionRequest;
      const AName: string; const ADefault: string = ''): string;

    // Abstract - Subclasses Must Implement
    procedure RegisterRoutes; virtual; abstract;
    function GetServiceInfo: TJSONObject; virtual;

  public
    constructor Create(const AServiceName: string; APort: Integer);
    destructor Destroy; override;

    procedure Start;
    procedure Stop;

    property ServiceName: string read FServiceName;
    property Port: Integer read FPort;
  end;

implementation

constructor TBaseMicroservice.Create(const AServiceName: string; APort: Integer);
begin
  inherited Create;
  FServiceName := AServiceName;
  FPort := APort;
  FLogger := TLogger.Create(AServiceName);
  FConfig := TAppConfiguration.GetInstance;

  FServer := TFPHTTPServer.Create(nil);
  FServer.Port := APort;
  FServer.OnRequest := @OnRequest;
  FServer.Threaded := True;
  FServer.AcceptIdleTimeout := 1000;

  SetupDefaultRoutes;
  RegisterRoutes;
end;

destructor TBaseMicroservice.Destroy;
begin
  FServer.Free;
  inherited Destroy;
end;

procedure TBaseMicroservice.SetupDefaultRoutes;
begin
  // Health Check Endpoints (Kubernetes/Docker)
  RegisterRoute(hmGet, '/health', @HandleHealth, False);
  RegisterRoute(hmGet, '/health/ready', @HandleReady, False);
  RegisterRoute(hmGet, '/health/live', @HandleHealth, False);
  RegisterRoute(hmGet, '/metrics', @HandleMetrics, False);
end;

procedure TBaseMicroservice.RegisterRoute(const AMethod: THttpMethod;
  const APath: string; const AHandler: TRouteHandler;
  ARequiresAuth: Boolean);
var
  Route: TRoute;
begin
  Route.Method := AMethod;
  Route.Path := APath;
  Route.Handler := AHandler;
  Route.RequiresAuth := ARequiresAuth;
  SetLength(FRoutes, Length(FRoutes) + 1);
  FRoutes[High(FRoutes)] := Route;
end;

procedure TBaseMicroservice.OnRequest(Sender: TObject;
  var ARequest: TFPHTTPConnectionRequest;
  var AResponse: TFPHTTPConnectionResponse);
var
  Method: THttpMethod;
  RouteIdx: Integer;
  Token: string;
  CorrelationId: string;
begin
  // CORS Headers
  HandleCors(AResponse);

  // Correlation ID (สำหรับ Distributed Tracing)
  CorrelationId := ARequest.GetHeader('X-Correlation-ID');
  if CorrelationId = '' then
    CorrelationId := TGuid.NewGuid.ToString;
  AResponse.SetHeader('X-Correlation-ID', CorrelationId);

  FLogger.Info(Format('[%s] %s %s',
    [CorrelationId, ARequest.Method, ARequest.PathInfo]));

  try
    Method := ParseMethod(ARequest.Method);
    RouteIdx := FindRoute(Method, ARequest.PathInfo);

    if RouteIdx < 0 then
    begin
      WriteError(AResponse, 'Route not found', 404);
      Exit;
    end;

    // Authentication Check
    if FRoutes[RouteIdx].RequiresAuth then
    begin
      Token := ARequest.GetHeader('Authorization');
      if not Token.StartsWith('Bearer ') then
      begin
        WriteError(AResponse, 'Unauthorized', 401);
        Exit;
      end;
      Token := Copy(Token, 8, MaxInt); // Remove "Bearer "
      if not ValidateToken(Token) then
      begin
        WriteError(AResponse, 'Invalid token', 401);
        Exit;
      end;
    end;

    // Call Route Handler
    FRoutes[RouteIdx].Handler(ARequest, AResponse);

  except
    on E: Exception do
    begin
      FLogger.Error('Unhandled error', E);
      WriteError(AResponse, 'Internal server error', 500);
    end;
  end;
end;

procedure TBaseMicroservice.WriteJson(AResponse: TFPHTTPConnectionResponse;
  const AJson: string; AStatusCode: Integer);
begin
  AResponse.Code := AStatusCode;
  AResponse.ContentType := 'application/json; charset=utf-8';
  AResponse.Content := AJson;
end;

procedure TBaseMicroservice.WriteError(AResponse: TFPHTTPConnectionResponse;
  const AMessage: string; AStatusCode: Integer);
var
  ErrorJson: TJSONObject;
begin
  ErrorJson := TJSONObject.Create;
  try
    ErrorJson.Add('error', AMessage);
    ErrorJson.Add('code', AStatusCode);
    ErrorJson.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', Now));
    WriteJson(AResponse, ErrorJson.AsJSON, AStatusCode);
  finally
    ErrorJson.Free;
  end;
end;

procedure TBaseMicroservice.HandleHealth(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  HealthJson: TJSONObject;
begin
  HealthJson := TJSONObject.Create;
  try
    HealthJson.Add('status', 'healthy');
    HealthJson.Add('service', FServiceName);
    HealthJson.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', Now));
    WriteJson(AResponse, HealthJson.AsJSON, 200);
  finally
    HealthJson.Free;
  end;
end;

procedure TBaseMicroservice.Start;
begin
  FLogger.Info(Format('%s starting on port %d', [FServiceName, FPort]));
  FServer.Active := True;
  FLogger.Info(FServiceName + ' started successfully');
end;

procedure TBaseMicroservice.Stop;
begin
  FLogger.Info(FServiceName + ' stopping...');
  FServer.Active := False;
  FLogger.Info(FServiceName + ' stopped');
end;

end.
```

## 3. Customer Service Implementation

```pascal
// CustomerService/uCustomerService.pas
unit uCustomerService;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpclient, httpdefs, fpjson,
  uBaseMicroservice, uCustomer, IRepository;

type
  TCustomerMicroservice = class(TBaseMicroservice)
  private
    FCustomerRepo: IRepository<TCustomer>;

    procedure HandleGetCustomers(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleGetCustomer(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleCreateCustomer(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleUpdateCustomer(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleDeleteCustomer(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure HandleSearchCustomers(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);

    function CustomerToJson(const ACustomer: TCustomer): TJSONObject;
    function JsonToCustomer(const AJson: TJSONObject): TCustomer;

  protected
    procedure RegisterRoutes; override;

  public
    constructor Create(APort: Integer = 8081);
  end;

implementation

constructor TCustomerMicroservice.Create(APort: Integer);
begin
  inherited Create('customer-service', APort);
  // Initialize Repository (from DI Container)
  FCustomerRepo := TDIContainer.Instance.Resolve<IRepository<TCustomer>>;
end;

procedure TCustomerMicroservice.RegisterRoutes;
begin
  // RESTful Routes
  RegisterRoute(hmGet,    '/api/v1/customers',     @HandleGetCustomers);
  RegisterRoute(hmPost,   '/api/v1/customers',     @HandleCreateCustomer);
  RegisterRoute(hmGet,    '/api/v1/customers/{id}',@HandleGetCustomer);
  RegisterRoute(hmPut,    '/api/v1/customers/{id}',@HandleUpdateCustomer);
  RegisterRoute(hmDelete, '/api/v1/customers/{id}',@HandleDeleteCustomer);
  RegisterRoute(hmGet,    '/api/v1/customers/search', @HandleSearchCustomers);
end;

procedure TCustomerMicroservice.HandleGetCustomers(
  ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  Customers: TList<TCustomer>;
  JsonArray: TJSONArray;
  Customer: TCustomer;
  ResponseObj: TJSONObject;
  Page, PageSize: Integer;
begin
  Page := StrToIntDef(GetQueryParam(ARequest, 'page', '1'), 1);
  PageSize := StrToIntDef(GetQueryParam(ARequest, 'pageSize', '20'), 20);

  // Limit page size
  if PageSize > 100 then PageSize := 100;

  Customers := FCustomerRepo.GetAll;
  try
    JsonArray := TJSONArray.Create;
    try
      for Customer in Customers do
        JsonArray.Add(CustomerToJson(Customer));

      ResponseObj := TJSONObject.Create;
      try
        ResponseObj.Add('data', JsonArray);
        ResponseObj.Add('page', Page);
        ResponseObj.Add('pageSize', PageSize);
        ResponseObj.Add('total', Customers.Count);
        WriteJson(AResponse, ResponseObj.AsJSON, 200);
      finally
        ResponseObj.Free;
      end;
    except
      JsonArray.Free;
      raise;
    end;
  finally
    Customers.Free;
  end;
end;

procedure TCustomerMicroservice.HandleGetCustomer(
  ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  CustomerId: Integer;
  Customer: TCustomer;
begin
  CustomerId := StrToIntDef(
    GetPathParam(ARequest.PathInfo, '/api/v1/customers/{id}', 'id'), 0);

  if CustomerId <= 0 then
  begin
    WriteError(AResponse, 'Invalid customer ID', 400);
    Exit;
  end;

  Customer := FCustomerRepo.GetById(CustomerId);
  if not Assigned(Customer) then
  begin
    WriteError(AResponse, 'Customer not found', 404);
    Exit;
  end;

  try
    WriteJson(AResponse, CustomerToJson(Customer).AsJSON, 200);
  finally
    Customer.Free;
  end;
end;

procedure TCustomerMicroservice.HandleCreateCustomer(
  ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  JsonBody: TJSONObject;
  Customer: TCustomer;
begin
  JsonBody := ReadJsonBody(ARequest);
  if not Assigned(JsonBody) then
  begin
    WriteError(AResponse, 'Invalid JSON body', 400);
    Exit;
  end;

  try
    Customer := JsonToCustomer(JsonBody);
    try
      FCustomerRepo.Add(Customer);
      WriteJson(AResponse, CustomerToJson(Customer).AsJSON, 201);
    except
      Customer.Free;
      raise;
    end;
  finally
    JsonBody.Free;
  end;
end;

function TCustomerMicroservice.CustomerToJson(
  const ACustomer: TCustomer): TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('id', ACustomer.Id);
  Result.Add('firstName', ACustomer.FirstName);
  Result.Add('lastName', ACustomer.LastName);
  Result.Add('fullName', ACustomer.FullName);
  Result.Add('email', ACustomer.Email);
  Result.Add('phone', ACustomer.Phone);
  Result.Add('status', Ord(ACustomer.Status));
  Result.Add('createdAt',
    FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', ACustomer.CreatedAt));
end;

end.
```

## 4. Service Discovery

```pascal
// uServiceDiscovery.pas - Service Registry Client
unit uServiceDiscovery;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpclient, fpjson, jsonparser,
  Generics.Collections, SyncObjs;

type
  TServiceInstance = record
    ServiceName: string;
    Host: string;
    Port: Integer;
    InstanceId: string;
    Status: string;
    LastHeartbeat: TDateTime;
    Tags: TArray<string>;
  end;

  IServiceDiscovery = interface
    // Registration
    procedure Register(const AServiceName, AHost: string; APort: Integer;
      const ATags: TArray<string>);
    procedure Deregister;
    procedure Heartbeat;

    // Discovery
    function GetInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetHealthyInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetLoadBalancedInstance(const AServiceName: string): TServiceInstance;
  end;

  // Consul-based Service Discovery
  TConsulServiceDiscovery = class(TInterfacedObject, IServiceDiscovery)
  private
    FConsulHost: string;
    FConsulPort: Integer;
    FServiceId: string;
    FHttpClient: TFPHTTPClient;
    FHeartbeatThread: TThread;
    FCriticalSection: TCriticalSection;
    FRoundRobinCounter: TDictionary<string, Integer>;

    function BuildUrl(const APath: string): string;
    function GetFromConsul(const APath: string): TJSONData;
    function PutToConsul(const APath, ABody: string): Boolean;
    function DeleteFromConsul(const APath: string): Boolean;

  public
    constructor Create(const AConsulHost: string = 'localhost';
      AConsulPort: Integer = 8500);
    destructor Destroy; override;

    procedure Register(const AServiceName, AHost: string; APort: Integer;
      const ATags: TArray<string>);
    procedure Deregister;
    procedure Heartbeat;

    function GetInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetHealthyInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetLoadBalancedInstance(const AServiceName: string): TServiceInstance;
  end;

  // In-Memory Service Discovery (สำหรับ Testing)
  TInMemoryServiceDiscovery = class(TInterfacedObject, IServiceDiscovery)
  private
    FInstances: TDictionary<string, TList<TServiceInstance>>;
    FCriticalSection: TCriticalSection;

  public
    constructor Create;
    destructor Destroy; override;

    procedure Register(const AServiceName, AHost: string; APort: Integer;
      const ATags: TArray<string>);
    procedure Deregister;
    procedure Heartbeat;

    function GetInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetHealthyInstances(const AServiceName: string): TArray<TServiceInstance>;
    function GetLoadBalancedInstance(const AServiceName: string): TServiceInstance;
  end;

implementation

constructor TConsulServiceDiscovery.Create(
  const AConsulHost: string; AConsulPort: Integer);
begin
  inherited Create;
  FConsulHost := AConsulHost;
  FConsulPort := AConsulPort;
  FHttpClient := TFPHTTPClient.Create(nil);
  FHttpClient.AddHeader('Content-Type', 'application/json');
  FCriticalSection := TCriticalSection.Create;
  FRoundRobinCounter := TDictionary<string, Integer>.Create;
end;

destructor TConsulServiceDiscovery.Destroy;
begin
  if FServiceId <> '' then
    Deregister;
  FRoundRobinCounter.Free;
  FCriticalSection.Free;
  FHttpClient.Free;
  inherited Destroy;
end;

function TConsulServiceDiscovery.BuildUrl(const APath: string): string;
begin
  Result := Format('http://%s:%d/v1/%s', [FConsulHost, FConsulPort, APath]);
end;

procedure TConsulServiceDiscovery.Register(
  const AServiceName, AHost: string; APort: Integer;
  const ATags: TArray<string>);
var
  Registration: TJSONObject;
  TagsArray: TJSONArray;
  Tag: string;
  Body: string;
begin
  FServiceId := Format('%s-%s-%d', [AServiceName, AHost, APort]);

  Registration := TJSONObject.Create;
  try
    Registration.Add('ID', FServiceId);
    Registration.Add('Name', AServiceName);
    Registration.Add('Address', AHost);
    Registration.Add('Port', APort);

    TagsArray := TJSONArray.Create;
    for Tag in ATags do
      TagsArray.Add(Tag);
    Registration.Add('Tags', TagsArray);

    // Health Check
    var HealthCheck := TJSONObject.Create;
    HealthCheck.Add('HTTP', Format('http://%s:%d/health', [AHost, APort]));
    HealthCheck.Add('Interval', '10s');
    HealthCheck.Add('Timeout', '5s');
    HealthCheck.Add('DeregisterCriticalServiceAfter', '1m');
    Registration.Add('Check', HealthCheck);

    Body := Registration.AsJSON;
    PutToConsul('agent/service/register', Body);
  finally
    Registration.Free;
  end;
end;

procedure TConsulServiceDiscovery.Deregister;
begin
  if FServiceId <> '' then
  begin
    PutToConsul('agent/service/deregister/' + FServiceId, '');
    FServiceId := '';
  end;
end;

function TConsulServiceDiscovery.GetHealthyInstances(
  const AServiceName: string): TArray<TServiceInstance>;
var
  JsonData: TJSONData;
  JsonArray: TJSONArray;
  ServiceObj, ChecksObj: TJSONObject;
  I: Integer;
  Instance: TServiceInstance;
  Instances: TList<TServiceInstance>;
  IsHealthy: Boolean;
begin
  JsonData := GetFromConsul('health/service/' + AServiceName + '?passing=true');
  Instances := TList<TServiceInstance>.Create;
  try
    if Assigned(JsonData) and (JsonData is TJSONArray) then
    begin
      JsonArray := TJSONArray(JsonData);
      for I := 0 to JsonArray.Count - 1 do
      begin
        ServiceObj := TJSONObject(TJSONObject(JsonArray[I]).Find('Service'));
        if Assigned(ServiceObj) then
        begin
          Instance.ServiceName := AServiceName;
          Instance.Host := ServiceObj.Get('Address', '');
          Instance.Port := ServiceObj.Get('Port', 0);
          Instance.InstanceId := ServiceObj.Get('ID', '');
          Instance.Status := 'healthy';
          Instance.LastHeartbeat := Now;
          Instances.Add(Instance);
        end;
      end;
    end;

    Result := Instances.ToArray;
  finally
    Instances.Free;
    if Assigned(JsonData) then JsonData.Free;
  end;
end;

function TConsulServiceDiscovery.GetLoadBalancedInstance(
  const AServiceName: string): TServiceInstance;
var
  Instances: TArray<TServiceInstance>;
  Counter: Integer;
begin
  Instances := GetHealthyInstances(AServiceName);
  if Length(Instances) = 0 then
    raise Exception.CreateFmt('No healthy instances for service: %s', [AServiceName]);

  FCriticalSection.Acquire;
  try
    if not FRoundRobinCounter.TryGetValue(AServiceName, Counter) then
      Counter := 0;

    Result := Instances[Counter mod Length(Instances)];
    FRoundRobinCounter.AddOrSetValue(AServiceName, Counter + 1);
  finally
    FCriticalSection.Release;
  end;
end;

end.
```

## 5. API Gateway

```pascal
// ApiGateway/uApiGateway.pas
unit uApiGateway;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpserver, fphttpclient, httpdefs,
  fpjson, SyncObjs, Generics.Collections,
  uBaseMicroservice, uServiceDiscovery, uJwtAuth;

type
  TProxyRoute = record
    PathPrefix: string;
    ServiceName: string;
    StripPrefix: Boolean;
    RequiresAuth: Boolean;
    RateLimit: Integer; // requests per minute
  end;

  TRateLimiter = class
  private
    FRequests: TDictionary<string, TList<TDateTime>>;
    FCriticalSection: TCriticalSection;

  public
    constructor Create;
    destructor Destroy; override;

    function IsAllowed(const AClientId: string;
      AMaxRequests: Integer; AWindowSeconds: Integer = 60): Boolean;
    procedure Cleanup;
  end;

  TApiGateway = class(TBaseMicroservice)
  private
    FServiceDiscovery: IServiceDiscovery;
    FJwtAuth: TJwtAuth;
    FProxyRoutes: array of TProxyRoute;
    FRateLimiter: TRateLimiter;

    procedure HandleProxy(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    function FindProxyRoute(const APath: string): Integer;
    function ForwardRequest(const AServiceName: string;
      ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse): Boolean;
    procedure AddProxyRoute(const APathPrefix, AServiceName: string;
      AStripPrefix: Boolean = True; ARequiresAuth: Boolean = True;
      ARateLimit: Integer = 1000);
    function GetClientId(ARequest: TFPHTTPConnectionRequest): string;

  protected
    procedure RegisterRoutes; override;

  public
    constructor Create(APort: Integer = 8080);
    destructor Destroy; override;
  end;

implementation

constructor TApiGateway.Create(APort: Integer);
begin
  inherited Create('api-gateway', APort);
  FServiceDiscovery := TConsulServiceDiscovery.Create;
  FJwtAuth := TJwtAuth.Create;
  FRateLimiter := TRateLimiter.Create;

  // Configure Proxy Routes
  AddProxyRoute('/api/customers', 'customer-service', True, True, 100);
  AddProxyRoute('/api/orders', 'order-service', True, True, 200);
  AddProxyRoute('/api/products', 'product-service', True, False, 500);
  AddProxyRoute('/api/payments', 'payment-service', True, True, 50);
end;

destructor TApiGateway.Destroy;
begin
  FRateLimiter.Free;
  FJwtAuth.Free;
  inherited Destroy;
end;

procedure TApiGateway.AddProxyRoute(const APathPrefix, AServiceName: string;
  AStripPrefix: Boolean; ARequiresAuth: Boolean; ARateLimit: Integer);
var
  Route: TProxyRoute;
begin
  Route.PathPrefix := APathPrefix;
  Route.ServiceName := AServiceName;
  Route.StripPrefix := AStripPrefix;
  Route.RequiresAuth := ARequiresAuth;
  Route.RateLimit := ARateLimit;
  SetLength(FProxyRoutes, Length(FProxyRoutes) + 1);
  FProxyRoutes[High(FProxyRoutes)] := Route;
end;

procedure TApiGateway.HandleProxy(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  RouteIdx: Integer;
  ClientId: string;
begin
  RouteIdx := FindProxyRoute(ARequest.PathInfo);
  if RouteIdx < 0 then
  begin
    WriteError(AResponse, 'Service not found', 404);
    Exit;
  end;

  // Rate Limiting
  ClientId := GetClientId(ARequest);
  if not FRateLimiter.IsAllowed(
    ClientId + ':' + FProxyRoutes[RouteIdx].ServiceName,
    FProxyRoutes[RouteIdx].RateLimit) then
  begin
    WriteError(AResponse, 'Rate limit exceeded', 429);
    Exit;
  end;

  // Authentication
  if FProxyRoutes[RouteIdx].RequiresAuth then
  begin
    var Token := ARequest.GetHeader('Authorization');
    if not FJwtAuth.ValidateToken(Token) then
    begin
      WriteError(AResponse, 'Unauthorized', 401);
      Exit;
    end;
  end;

  // Forward Request
  if not ForwardRequest(FProxyRoutes[RouteIdx].ServiceName, ARequest, AResponse) then
    WriteError(AResponse, 'Service unavailable', 503);
end;

function TApiGateway.ForwardRequest(const AServiceName: string;
  ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse): Boolean;
var
  Instance: TServiceInstance;
  Client: TFPHTTPClient;
  TargetUrl: string;
  ResponseStream: TStringStream;
begin
  Result := False;
  try
    Instance := FServiceDiscovery.GetLoadBalancedInstance(AServiceName);

    TargetUrl := Format('http://%s:%d%s',
      [Instance.Host, Instance.Port, ARequest.PathInfo]);
    if ARequest.QueryString <> '' then
      TargetUrl := TargetUrl + '?' + ARequest.QueryString;

    Client := TFPHTTPClient.Create(nil);
    ResponseStream := TStringStream.Create('', CP_UTF8);
    try
      // Forward Headers
      Client.AddHeader('X-Forwarded-For', ARequest.RemoteAddress);
      Client.AddHeader('X-Forwarded-Host', ARequest.Host);
      Client.AddHeader('X-Correlation-ID', ARequest.GetHeader('X-Correlation-ID'));

      // Forward Auth Header if present
      if ARequest.GetHeader('Authorization') <> '' then
        Client.AddHeader('Authorization', ARequest.GetHeader('Authorization'));

      // Make Request
      case ParseMethod(ARequest.Method) of
        hmGet: Client.Get(TargetUrl, ResponseStream);
        hmPost: Client.FormPost(TargetUrl, ARequest.Content, ResponseStream);
        hmPut: Client.Put(TargetUrl, ARequest.Content, ResponseStream);
        hmDelete: Client.Delete(TargetUrl, ResponseStream);
      end;

      AResponse.Code := Client.ResponseStatusCode;
      AResponse.ContentType := Client.ResponseHeaders.Values['Content-Type'];
      AResponse.Content := ResponseStream.DataString;

      Result := True;
    finally
      ResponseStream.Free;
      Client.Free;
    end;
  except
    on E: Exception do
    begin
      FLogger.Error('Proxy error', E);
      Result := False;
    end;
  end;
end;

// Rate Limiter Implementation
constructor TRateLimiter.Create;
begin
  inherited Create;
  FRequests := TDictionary<string, TList<TDateTime>>.Create;
  FCriticalSection := TCriticalSection.Create;
end;

destructor TRateLimiter.Destroy;
var
  List: TList<TDateTime>;
begin
  FCriticalSection.Free;
  for List in FRequests.Values do
    List.Free;
  FRequests.Free;
  inherited Destroy;
end;

function TRateLimiter.IsAllowed(const AClientId: string;
  AMaxRequests: Integer; AWindowSeconds: Integer): Boolean;
var
  Requests: TList<TDateTime>;
  Now_: TDateTime;
  WindowStart: TDateTime;
  I: Integer;
begin
  FCriticalSection.Acquire;
  try
    Now_ := Now;
    WindowStart := Now_ - (AWindowSeconds / SecsPerDay);

    if not FRequests.TryGetValue(AClientId, Requests) then
    begin
      Requests := TList<TDateTime>.Create;
      FRequests.Add(AClientId, Requests);
    end;

    // Remove old requests outside the window
    I := 0;
    while I < Requests.Count do
      if Requests[I] < WindowStart then
        Requests.Delete(I)
      else
        Inc(I);

    // Check if allowed
    if Requests.Count >= AMaxRequests then
    begin
      Result := False;
      Exit;
    end;

    Requests.Add(Now_);
    Result := True;
  finally
    FCriticalSection.Release;
  end;
end;

end.
```

## 6. Message Queue with RabbitMQ

```pascal
// uMessageQueue.pas - RabbitMQ Integration
unit uMessageQueue;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, Generics.Collections;

type
  TMessageHandler<T> = procedure(const AMessage: T) of object;

  IMessage = interface
    function GetCorrelationId: string;
    function GetTimestamp: TDateTime;
    function GetType: string;
    function Serialize: string;
  end;

  TBaseMessage = class(TInterfacedObject, IMessage)
  private
    FCorrelationId: string;
    FTimestamp: TDateTime;
    FType: string;
  public
    constructor Create(const AType: string);
    function GetCorrelationId: string;
    function GetTimestamp: TDateTime;
    function GetType: string;
    function Serialize: string; virtual; abstract;
  end;

  // Order Events
  TOrderCreatedMessage = class(TBaseMessage)
  public
    OrderId: Integer;
    CustomerId: Integer;
    TotalAmount: Double;
    Currency: string;
    Items: TArray<record ProductId, Quantity: Integer; Price: Double; end>;

    constructor Create(AOrderId, ACustomerId: Integer;
      ATotalAmount: Double; const ACurrency: string);
    function Serialize: string; override;
  end;

  TOrderStatusChangedMessage = class(TBaseMessage)
  public
    OrderId: Integer;
    OldStatus: string;
    NewStatus: string;
    Reason: string;

    constructor Create(AOrderId: Integer;
      const AOldStatus, ANewStatus, AReason: string);
    function Serialize: string; override;
  end;

  IMessageBus = interface
    // Publishing
    procedure Publish(const AExchange, ARoutingKey: string;
      const AMessage: IMessage);
    procedure PublishToQueue(const AQueueName: string;
      const AMessage: IMessage);

    // Subscribing
    procedure Subscribe(const AQueueName: string;
      const AHandler: TProc<string>);
    procedure SubscribeToExchange(const AExchange, AQueueName,
      ARoutingKey: string; const AHandler: TProc<string>);

    // Connection Management
    procedure Connect;
    procedure Disconnect;
    function IsConnected: Boolean;
  end;

  // RabbitMQ-based Message Bus
  TRabbitMQBus = class(TInterfacedObject, IMessageBus)
  private
    FHost: string;
    FPort: Integer;
    FUsername: string;
    FPassword: string;
    FVirtualHost: string;
    FConnected: Boolean;
    // AMQP Connection (using external library)
    // FChannel: TAMQPChannel;

    procedure EnsureConnected;

  public
    constructor Create(const AHost: string = 'localhost';
      APort: Integer = 5672;
      const AUsername: string = 'guest';
      const APassword: string = 'guest';
      const AVirtualHost: string = '/');
    destructor Destroy; override;

    procedure Publish(const AExchange, ARoutingKey: string;
      const AMessage: IMessage);
    procedure PublishToQueue(const AQueueName: string;
      const AMessage: IMessage);
    procedure Subscribe(const AQueueName: string;
      const AHandler: TProc<string>);
    procedure SubscribeToExchange(const AExchange, AQueueName,
      ARoutingKey: string; const AHandler: TProc<string>);

    procedure Connect;
    procedure Disconnect;
    function IsConnected: Boolean;
  end;

  // In-Process Event Bus (สำหรับ Testing)
  TInMemoryMessageBus = class(TInterfacedObject, IMessageBus)
  private
    FHandlers: TDictionary<string, TList<TProc<string>>>;
    FCriticalSection: TCriticalSection;

  public
    constructor Create;
    destructor Destroy; override;

    procedure Publish(const AExchange, ARoutingKey: string;
      const AMessage: IMessage);
    procedure PublishToQueue(const AQueueName: string;
      const AMessage: IMessage);
    procedure Subscribe(const AQueueName: string;
      const AHandler: TProc<string>);
    procedure SubscribeToExchange(const AExchange, AQueueName,
      ARoutingKey: string; const AHandler: TProc<string>);

    procedure Connect;
    procedure Disconnect;
    function IsConnected: Boolean;
  end;

implementation

constructor TBaseMessage.Create(const AType: string);
begin
  inherited Create;
  FType := AType;
  FCorrelationId := TGuid.NewGuid.ToString;
  FTimestamp := Now;
end;

constructor TOrderCreatedMessage.Create(AOrderId, ACustomerId: Integer;
  ATotalAmount: Double; const ACurrency: string);
begin
  inherited Create('order.created');
  OrderId := AOrderId;
  CustomerId := ACustomerId;
  TotalAmount := ATotalAmount;
  Currency := ACurrency;
end;

function TOrderCreatedMessage.Serialize: string;
var
  Json: TJSONObject;
begin
  Json := TJSONObject.Create;
  try
    Json.Add('correlationId', GetCorrelationId);
    Json.Add('type', GetType);
    Json.Add('timestamp', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss"Z"', GetTimestamp));
    Json.Add('orderId', OrderId);
    Json.Add('customerId', CustomerId);
    Json.Add('totalAmount', TotalAmount);
    Json.Add('currency', Currency);
    Result := Json.AsJSON;
  finally
    Json.Free;
  end;
end;

// In-Memory Bus for Testing
constructor TInMemoryMessageBus.Create;
begin
  inherited Create;
  FHandlers := TDictionary<string, TList<TProc<string>>>.Create;
  FCriticalSection := TCriticalSection.Create;
end;

destructor TInMemoryMessageBus.Destroy;
var
  HandlerList: TList<TProc<string>>;
begin
  FCriticalSection.Free;
  for HandlerList in FHandlers.Values do
    HandlerList.Free;
  FHandlers.Free;
  inherited Destroy;
end;

procedure TInMemoryMessageBus.Subscribe(const AQueueName: string;
  const AHandler: TProc<string>);
var
  HandlerList: TList<TProc<string>>;
begin
  FCriticalSection.Acquire;
  try
    if not FHandlers.TryGetValue(AQueueName, HandlerList) then
    begin
      HandlerList := TList<TProc<string>>.Create;
      FHandlers.Add(AQueueName, HandlerList);
    end;
    HandlerList.Add(AHandler);
  finally
    FCriticalSection.Release;
  end;
end;

procedure TInMemoryMessageBus.PublishToQueue(const AQueueName: string;
  const AMessage: IMessage);
var
  HandlerList: TList<TProc<string>>;
  Handler: TProc<string>;
  MessageBody: string;
begin
  MessageBody := AMessage.Serialize;

  FCriticalSection.Acquire;
  try
    if FHandlers.TryGetValue(AQueueName, HandlerList) then
      for Handler in HandlerList do
        Handler(MessageBody);
  finally
    FCriticalSection.Release;
  end;
end;

procedure TInMemoryMessageBus.Connect;
begin
  FConnected := True;
end;

function TInMemoryMessageBus.IsConnected: Boolean;
begin
  Result := FConnected;
end;

end.
```

## 7. Inter-Service Communication with Circuit Breaker

```pascal
// uServiceClient.pas - Service-to-Service Communication
unit uServiceClient;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpclient, fpjson, Generics.Collections,
  uServiceDiscovery, uCircuitBreaker;

type
  TServiceClient = class
  private
    FServiceDiscovery: IServiceDiscovery;
    FCircuitBreakers: TDictionary<string, TCircuitBreaker>;
    FDefaultTimeout: Integer;

    function GetCircuitBreaker(const AServiceName: string): TCircuitBreaker;

  public
    constructor Create(const AServiceDiscovery: IServiceDiscovery;
      ADefaultTimeout: Integer = 5000);
    destructor Destroy; override;

    function Get(const AServiceName, APath: string;
      const AHeaders: TStrings = nil): TJSONData;
    function Post(const AServiceName, APath: string;
      const ABody: string; const AHeaders: TStrings = nil): TJSONData;
    function Put(const AServiceName, APath: string;
      const ABody: string): TJSONData;
    function Delete(const AServiceName, APath: string): Boolean;
  end;

  TCircuitBreakerState = (cbsClosed, cbsOpen, cbsHalfOpen);

  TCircuitBreaker = class
  private
    FServiceName: string;
    FState: TCircuitBreakerState;
    FFailureCount: Integer;
    FSuccessCount: Integer;
    FFailureThreshold: Integer;
    FSuccessThreshold: Integer;
    FOpenTimeout: Integer; // milliseconds
    FLastFailureTime: TDateTime;

    procedure RecordSuccess;
    procedure RecordFailure;
    function CanAttempt: Boolean;

  public
    constructor Create(const AServiceName: string;
      AFailureThreshold: Integer = 5;
      ASuccessThreshold: Integer = 2;
      AOpenTimeout: Integer = 60000);

    function Execute<T>(const AFunc: TFunc<T>): T;
    procedure Reset;

    property State: TCircuitBreakerState read FState;
    property ServiceName: string read FServiceName;
  end;

implementation

constructor TCircuitBreaker.Create(const AServiceName: string;
  AFailureThreshold, ASuccessThreshold, AOpenTimeout: Integer);
begin
  inherited Create;
  FServiceName := AServiceName;
  FState := cbsClosed;
  FFailureThreshold := AFailureThreshold;
  FSuccessThreshold := ASuccessThreshold;
  FOpenTimeout := AOpenTimeout;
  FFailureCount := 0;
  FSuccessCount := 0;
end;

function TCircuitBreaker.Execute<T>(const AFunc: TFunc<T>): T;
begin
  if not CanAttempt then
    raise ECircuitBreakerOpenException.CreateFmt(
      'Circuit breaker for %s is OPEN', [FServiceName]);

  try
    Result := AFunc();
    RecordSuccess;
  except
    RecordFailure;
    raise;
  end;
end;

function TCircuitBreaker.CanAttempt: Boolean;
var
  TimeSinceLastFailure: Int64;
begin
  case FState of
    cbsClosed: Result := True;
    cbsOpen:
    begin
      // Check if timeout has passed (Half-Open)
      TimeSinceLastFailure := Round((Now - FLastFailureTime) * MSecsPerDay);
      if TimeSinceLastFailure >= FOpenTimeout then
      begin
        FState := cbsHalfOpen;
        FSuccessCount := 0;
        Result := True;
      end
      else
        Result := False;
    end;
    cbsHalfOpen: Result := True;
  end;
end;

procedure TCircuitBreaker.RecordSuccess;
begin
  FFailureCount := 0;
  if FState = cbsHalfOpen then
  begin
    Inc(FSuccessCount);
    if FSuccessCount >= FSuccessThreshold then
    begin
      FState := cbsClosed;
      FSuccessCount := 0;
    end;
  end;
end;

procedure TCircuitBreaker.RecordFailure;
begin
  FLastFailureTime := Now;
  Inc(FFailureCount);
  if FState in [cbsClosed, cbsHalfOpen] then
  begin
    if FFailureCount >= FFailureThreshold then
    begin
      FState := cbsOpen;
      FFailureCount := 0;
    end;
  end;
end;

procedure TCircuitBreaker.Reset;
begin
  FState := cbsClosed;
  FFailureCount := 0;
  FSuccessCount := 0;
end;

// Service Client
constructor TServiceClient.Create(
  const AServiceDiscovery: IServiceDiscovery; ADefaultTimeout: Integer);
begin
  inherited Create;
  FServiceDiscovery := AServiceDiscovery;
  FDefaultTimeout := ADefaultTimeout;
  FCircuitBreakers := TDictionary<string, TCircuitBreaker>.Create;
end;

destructor TServiceClient.Destroy;
var
  CB: TCircuitBreaker;
begin
  for CB in FCircuitBreakers.Values do
    CB.Free;
  FCircuitBreakers.Free;
  inherited Destroy;
end;

function TServiceClient.GetCircuitBreaker(
  const AServiceName: string): TCircuitBreaker;
begin
  if not FCircuitBreakers.TryGetValue(AServiceName, Result) then
  begin
    Result := TCircuitBreaker.Create(AServiceName);
    FCircuitBreakers.Add(AServiceName, Result);
  end;
end;

function TServiceClient.Get(const AServiceName, APath: string;
  const AHeaders: TStrings): TJSONData;
var
  CB: TCircuitBreaker;
begin
  CB := GetCircuitBreaker(AServiceName);
  Result := CB.Execute<TJSONData>(
    function: TJSONData
    var
      Instance: TServiceInstance;
      Client: TFPHTTPClient;
      ResponseStr: TStringStream;
      Url: string;
    begin
      Instance := FServiceDiscovery.GetLoadBalancedInstance(AServiceName);
      Url := Format('http://%s:%d%s', [Instance.Host, Instance.Port, APath]);

      Client := TFPHTTPClient.Create(nil);
      ResponseStr := TStringStream.Create;
      try
        Client.ConnectTimeout := FDefaultTimeout;
        Client.IOTimeout := FDefaultTimeout;

        if Assigned(AHeaders) then
        begin
          var I: Integer;
          for I := 0 to AHeaders.Count - 1 do
            Client.AddHeader(AHeaders.Names[I], AHeaders.ValueFromIndex[I]);
        end;

        Client.Get(Url, ResponseStr);

        if Client.ResponseStatusCode >= 400 then
          raise EServiceCallException.CreateFmt(
            'Service %s returned %d', [AServiceName, Client.ResponseStatusCode]);

        Result := GetJSON(ResponseStr.DataString);
      finally
        ResponseStr.Free;
        Client.Free;
      end;
    end
  );
end;

end.
```

## 8. Microservice Main Program

```pascal
// OrderService/OrderService.pas - Order Microservice Main
program OrderService;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uOrderMicroservice,
  uDIContainer, uAppStartup,
  uConfiguration, uLogger;

var
  Service: TOrderMicroservice;
  Logger: ILogger;

procedure HandleShutdown(Signal: Integer); cdecl;
begin
  Logger.Info('Received shutdown signal, stopping service...');
  if Assigned(Service) then
    Service.Stop;
end;

begin
  // Initialize Configuration
  TAppConfiguration.Initialize;

  Logger := TLogger.Create('order-service');
  Logger.AddSink(TConsoleSink.Create);
  Logger.AddSink(TFileSink.Create('/var/log/order-service/app.log'));

  Logger.Info('Order Service starting...');
  Logger.Info(Format('Environment: %s', [TAppConfiguration.GetInstance.Environment]));

  // Setup Signal Handlers (สำหรับ Graceful Shutdown)
  {$IFDEF UNIX}
  fpSignal(SIGTERM, @HandleShutdown);
  fpSignal(SIGINT, @HandleShutdown);
  {$ENDIF}

  // Configure DI Container
  TDIContainer.Instance.RegisterSingleton<IServiceDiscovery, TConsulServiceDiscovery>;
  TDIContainer.Instance.RegisterSingleton<IMessageBus, TRabbitMQBus>;

  // Create and Start Service
  Service := TOrderMicroservice.Create(
    StrToIntDef(GetEnvironmentVariable('PORT'), 8082)
  );
  try
    Service.Start;
    Logger.Info(Format('Order Service listening on port %d', [Service.Port]));

    // Register with Service Discovery
    TDIContainer.Instance.Resolve<IServiceDiscovery>.Register(
      'order-service',
      GetEnvironmentVariable('HOST'),
      Service.Port,
      ['v1', 'orders']
    );

    // Keep running until shutdown
    while Service.IsRunning do
      Sleep(1000);

  finally
    Service.Free;
    Logger.Info('Order Service stopped');
  end;
end.
```

## 9. Docker Compose Setup

```yaml
# docker-compose.yml - Microservices Setup
# (ไฟล์ YAML สำหรับ reference เท่านั้น)
version: '3.8'

services:
  # Infrastructure Services
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_USER: appuser
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

  consul:
    image: consul:latest
    ports:
      - "8500:8500"

  # Microservices
  api-gateway:
    build: ./ApiGateway
    ports:
      - "8080:8080"
    environment:
      - CONSUL_HOST=consul
      - PORT=8080

  customer-service:
    build: ./CustomerService
    environment:
      - DB_HOST=postgres
      - DB_NAME=customers
      - RABBITMQ_HOST=rabbitmq
      - CONSUL_HOST=consul
      - PORT=8081

  order-service:
    build: ./OrderService
    environment:
      - DB_HOST=postgres
      - DB_NAME=orders
      - RABBITMQ_HOST=rabbitmq
      - CONSUL_HOST=consul
      - PORT=8082
```

## 10. สรุปบทเรียน

Microservices Architecture ให้ประโยชน์หลายอย่าง:

**ข้อดี:**
- Scale แต่ละ Service ได้อิสระ
- Deploy แต่ละ Service ได้อิสระ
- ทีมต่างๆ ทำงานได้อิสระ
- เลือก Technology ที่เหมาะสมกับแต่ละ Service ได้

**ข้อเสีย:**
- ซับซ้อนกว่า Monolith
- ต้องจัดการ Network Latency
- Debugging ยากขึ้น
- ต้องการ DevOps ที่แข็งแกร่ง

**Pattern สำคัญที่ได้เรียน:**
1. API Gateway Pattern
2. Service Discovery
3. Circuit Breaker Pattern
4. Message Queue (Async Communication)
5. Health Check Endpoints
6. Rate Limiting
