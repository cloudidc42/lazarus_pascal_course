# ตอนที่ 71: Docker Integration ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้าง Docker image สำหรับแอปพลิเคชัน Pascal, การใช้ Docker Compose และการ deploy

---

## 71.1 Dockerfile สำหรับ Pascal

```dockerfile
# Dockerfile สำหรับ build Pascal/Free Pascal application

# === Stage 1: Build ===
FROM debian:bookworm-slim AS builder

# ติดตั้ง Free Pascal Compiler
RUN apt-get update && apt-get install -y \
    fpc \
    libx11-dev \
    libgtk2.0-dev \
    && rm -rf /var/lib/apt/lists/*

# ตั้งค่า working directory
WORKDIR /app

# Copy source code
COPY src/ .

# Build executable
RUN fpc -O2 -dRELEASE main.pas -o /app/myapp

# === Stage 2: Runtime ===
FROM debian:bookworm-slim

# ติดตั้ง runtime dependencies
RUN apt-get update && apt-get install -y \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# สร้าง non-root user
RUN useradd -r -s /bin/false appuser

# Copy binary
COPY --from=builder /app/myapp /usr/local/bin/myapp
RUN chmod +x /usr/local/bin/myapp

# ตั้งค่า user
USER appuser

# Expose port ถ้าเป็น web server
EXPOSE 8080

# เริ่มต้น application
CMD ["/usr/local/bin/myapp"]
```

---

## 71.2 Pascal HTTP Server สำหรับ Docker

```pascal
unit http_server;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Sockets, BaseUnix, Unix;

type
  THTTPRequest = record
    Method: string;
    Path: string;
    Version: string;
    Headers: TStringList;
    Body: string;
    QueryString: string;
    QueryParams: TStringList;
  end;

  THTTPResponse = record
    StatusCode: Integer;
    StatusText: string;
    Headers: TStringList;
    Body: string;
    ContentType: string;
  end;

  TRequestHandler = procedure(const ARequest: THTTPRequest; 
    var AResponse: THTTPResponse) of object;

  THTTPRoute = record
    Method: string;
    Path: string;
    Handler: TRequestHandler;
  end;

  TSimpleHTTPServer = class
  private
    FSocket: Integer;
    FPort: Integer;
    FRunning: Boolean;
    FRoutes: array of THTTPRoute;
    FRouteCount: Integer;
    
    function ParseRequest(const ARawRequest: string; out ARequest: THTTPRequest): Boolean;
    procedure BuildResponse(const AResponse: THTTPResponse; out ARaw: string);
    procedure HandleConnection(AClientSocket: Integer);
    function FindRoute(const AMethod, APath: string): Integer;
    procedure ParseQueryString(const AQuery: string; AParams: TStringList);
    procedure DefaultNotFound(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
    
  public
    constructor Create(APort: Integer = 8080);
    destructor Destroy; override;
    
    procedure AddRoute(const AMethod, APath: string; AHandler: TRequestHandler);
    procedure Get(const APath: string; AHandler: TRequestHandler);
    procedure Post(const APath: string; AHandler: TRequestHandler);
    
    procedure Start;
    procedure Stop;
    
    class function OKResponse(const ABody: string; 
      const AContentType: string = 'application/json'): THTTPResponse;
    class function ErrorResponse(ACode: Integer; const AMessage: string): THTTPResponse;
    
    property Port: Integer read FPort;
    property Running: Boolean read FRunning;
  end;

implementation

{ TSimpleHTTPServer }

constructor TSimpleHTTPServer.Create(APort: Integer);
begin
  inherited Create;
  FPort := APort;
  FRunning := False;
  FRouteCount := 0;
  SetLength(FRoutes, 100);
end;

destructor TSimpleHTTPServer.Destroy;
begin
  Stop;
  inherited Destroy;
end;

procedure TSimpleHTTPServer.AddRoute(const AMethod, APath: string; 
  AHandler: TRequestHandler);
begin
  if FRouteCount < Length(FRoutes) then
  begin
    FRoutes[FRouteCount].Method := UpperCase(AMethod);
    FRoutes[FRouteCount].Path := APath;
    FRoutes[FRouteCount].Handler := AHandler;
    Inc(FRouteCount);
  end;
end;

procedure TSimpleHTTPServer.Get(const APath: string; AHandler: TRequestHandler);
begin
  AddRoute('GET', APath, AHandler);
end;

procedure TSimpleHTTPServer.Post(const APath: string; AHandler: TRequestHandler);
begin
  AddRoute('POST', APath, AHandler);
end;

function TSimpleHTTPServer.FindRoute(const AMethod, APath: string): Integer;
var
  i: Integer;
  RoutePath: string;
begin
  Result := -1;
  
  for i := 0 to FRouteCount - 1 do
  begin
    RoutePath := FRoutes[i].Path;
    
    if (FRoutes[i].Method = UpperCase(AMethod)) and
       ((APath = RoutePath) or
        // Simple wildcard matching
        ((Copy(RoutePath, Length(RoutePath), 1) = '*') and
         (Copy(APath, 1, Length(RoutePath) - 1) = Copy(RoutePath, 1, Length(RoutePath) - 1)))) then
    begin
      Result := i;
      Exit;
    end;
  end;
end;

procedure TSimpleHTTPServer.ParseQueryString(const AQuery: string; AParams: TStringList);
var
  Parts: TStringList;
  i: Integer;
begin
  AParams.Clear;
  
  Parts := TStringList.Create;
  try
    Parts.Delimiter := '&';
    Parts.DelimitedText := AQuery;
    
    for i := 0 to Parts.Count - 1 do
    begin
      var EqPos := Pos('=', Parts[i]);
      if EqPos > 0 then
        AParams.Values[Copy(Parts[i], 1, EqPos - 1)] := 
          Copy(Parts[i], EqPos + 1, MaxInt);
    end;
  finally
    Parts.Free;
  end;
end;

function TSimpleHTTPServer.ParseRequest(const ARawRequest: string; 
  out ARequest: THTTPRequest): Boolean;
var
  Lines: TStringList;
  FirstLine: string;
  Parts: TStringList;
  i: Integer;
  QPos: Integer;
begin
  Result := False;
  FillChar(ARequest, SizeOf(ARequest), 0);
  ARequest.Headers := TStringList.Create;
  ARequest.QueryParams := TStringList.Create;
  
  Lines := TStringList.Create;
  Parts := TStringList.Create;
  try
    Lines.Text := ARawRequest;
    
    if Lines.Count = 0 then Exit;
    
    // Parse request line: GET /path HTTP/1.1
    FirstLine := Lines[0];
    Parts.Delimiter := ' ';
    Parts.DelimitedText := FirstLine;
    
    if Parts.Count >= 3 then
    begin
      ARequest.Method := Parts[0];
      ARequest.Path := Parts[1];
      ARequest.Version := Parts[2];
      
      // Extract query string
      QPos := Pos('?', ARequest.Path);
      if QPos > 0 then
      begin
        ARequest.QueryString := Copy(ARequest.Path, QPos + 1, MaxInt);
        ARequest.Path := Copy(ARequest.Path, 1, QPos - 1);
        ParseQueryString(ARequest.QueryString, ARequest.QueryParams);
      end;
    end;
    
    // Parse headers
    i := 1;
    while (i < Lines.Count) and (Lines[i] <> '') do
    begin
      ARequest.Headers.Add(Lines[i]);
      Inc(i);
    end;
    
    // Skip blank line, rest is body
    Inc(i);
    if i < Lines.Count then
    begin
      var BodyStart := Pos(#13#10#13#10, ARawRequest);
      if BodyStart > 0 then
        ARequest.Body := Copy(ARawRequest, BodyStart + 4, MaxInt);
    end;
    
    Result := ARequest.Method <> '';
    
  finally
    Lines.Free;
    Parts.Free;
  end;
end;

procedure TSimpleHTTPServer.BuildResponse(const AResponse: THTTPResponse; 
  out ARaw: string);
var
  StatusTexts: array[200..599] of string;
  i: Integer;
begin
  StatusTexts[200] := 'OK';
  StatusTexts[201] := 'Created';
  StatusTexts[204] := 'No Content';
  StatusTexts[400] := 'Bad Request';
  StatusTexts[401] := 'Unauthorized';
  StatusTexts[403] := 'Forbidden';
  StatusTexts[404] := 'Not Found';
  StatusTexts[500] := 'Internal Server Error';
  
  var StatusText := AResponse.StatusText;
  if (StatusText = '') and (AResponse.StatusCode >= 200) and 
     (AResponse.StatusCode <= 599) then
    StatusText := StatusTexts[AResponse.StatusCode];
  
  ARaw := Format('HTTP/1.1 %d %s'#13#10, [AResponse.StatusCode, StatusText]);
  ARaw := ARaw + Format('Content-Length: %d'#13#10, [Length(AResponse.Body)]);
  ARaw := ARaw + Format('Content-Type: %s'#13#10, 
    [IfThen(AResponse.ContentType = '', 'application/json', AResponse.ContentType)]);
  ARaw := ARaw + 'Connection: close'#13#10;
  ARaw := ARaw + 'Server: Pascal/FPC'#13#10;
  
  if Assigned(AResponse.Headers) then
    for i := 0 to AResponse.Headers.Count - 1 do
      ARaw := ARaw + AResponse.Headers[i] + #13#10;
      
  ARaw := ARaw + #13#10;  // Blank line
  ARaw := ARaw + AResponse.Body;
end;

procedure TSimpleHTTPServer.DefaultNotFound(const ARequest: THTTPRequest; 
  var AResponse: THTTPResponse);
begin
  AResponse := ErrorResponse(404, 'Not Found: ' + ARequest.Path);
end;

procedure TSimpleHTTPServer.HandleConnection(AClientSocket: Integer);
var
  Buffer: array[0..4095] of AnsiChar;
  BytesRead: Integer;
  RawRequest: string;
  Request: THTTPRequest;
  Response: THTTPResponse;
  RawResponse: string;
  RouteIdx: Integer;
begin
  FillChar(Buffer, SizeOf(Buffer), 0);
  BytesRead := fpRecv(AClientSocket, @Buffer, SizeOf(Buffer) - 1, 0);
  
  if BytesRead > 0 then
  begin
    RawRequest := Copy(String(Buffer), 1, BytesRead);
    
    if ParseRequest(RawRequest, Request) then
    begin
      FillChar(Response, SizeOf(Response), 0);
      Response.Headers := TStringList.Create;
      
      try
        RouteIdx := FindRoute(Request.Method, Request.Path);
        
        if RouteIdx >= 0 then
          FRoutes[RouteIdx].Handler(Request, Response)
        else
          DefaultNotFound(Request, Response);
          
        BuildResponse(Response, RawResponse);
        fpSend(AClientSocket, @RawResponse[1], Length(RawResponse), 0);
        
      finally
        Response.Headers.Free;
        Request.Headers.Free;
        Request.QueryParams.Free;
      end;
    end;
  end;
  
  fpClose(AClientSocket);
end;

procedure TSimpleHTTPServer.Start;
var
  ServerAddr: TInetSockAddr;
  ClientAddr: TInetSockAddr;
  ClientLen: TSockLen;
  ClientSocket: Integer;
  ReuseAddr: Integer;
begin
  FSocket := fpSocket(AF_INET, SOCK_STREAM, 0);
  
  if FSocket < 0 then
  begin
    WriteLn('ไม่สามารถสร้าง socket');
    Exit;
  end;
  
  // Allow reuse address
  ReuseAddr := 1;
  fpSetSockOpt(FSocket, SOL_SOCKET, SO_REUSEADDR, @ReuseAddr, SizeOf(ReuseAddr));
  
  // Bind
  FillChar(ServerAddr, SizeOf(ServerAddr), 0);
  ServerAddr.sin_family := AF_INET;
  ServerAddr.sin_port := htons(FPort);
  ServerAddr.sin_addr.s_addr := INADDR_ANY;
  
  if fpBind(FSocket, @ServerAddr, SizeOf(ServerAddr)) < 0 then
  begin
    WriteLn('ไม่สามารถ bind port: ', FPort);
    fpClose(FSocket);
    Exit;
  end;
  
  fpListen(FSocket, 10);
  FRunning := True;
  
  WriteLn('Server started on port ', FPort);
  WriteLn('Press Ctrl+C to stop');
  
  ClientLen := SizeOf(ClientAddr);
  
  while FRunning do
  begin
    ClientSocket := fpAccept(FSocket, @ClientAddr, @ClientLen);
    
    if ClientSocket >= 0 then
      HandleConnection(ClientSocket);
  end;
end;

procedure TSimpleHTTPServer.Stop;
begin
  FRunning := False;
  if FSocket >= 0 then
    fpClose(FSocket);
end;

class function TSimpleHTTPServer.OKResponse(const ABody: string; 
  const AContentType: string): THTTPResponse;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.StatusCode := 200;
  Result.Body := ABody;
  Result.ContentType := AContentType;
end;

class function TSimpleHTTPServer.ErrorResponse(ACode: Integer; 
  const AMessage: string): THTTPResponse;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.StatusCode := ACode;
  Result.Body := Format('{"error": "%s", "code": %d}', [AMessage, ACode]);
  Result.ContentType := 'application/json';
end;

end.
```

---

## 71.3 ตัวอย่าง API Server

```pascal
program api_server;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  http_server;

type
  TAPIServer = class
  private
    FServer: TSimpleHTTPServer;
    FUsers: TStringList;  // Simple user store
    
    procedure HandleHealth(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
    procedure HandleGetUsers(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
    procedure HandleCreateUser(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
    procedure HandleGetUser(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
    
  public
    constructor Create(APort: Integer);
    destructor Destroy; override;
    procedure Run;
  end;

constructor TAPIServer.Create(APort: Integer);
begin
  inherited Create;
  FUsers := TStringList.Create;
  
  // Sample data
  FUsers.Add('1:สมชาย ใจดี:somchai@email.com');
  FUsers.Add('2:สมหญิง รักเรียน:somying@email.com');
  
  FServer := TSimpleHTTPServer.Create(APort);
  
  // Routes
  FServer.Get('/health', @HandleHealth);
  FServer.Get('/api/users', @HandleGetUsers);
  FServer.Post('/api/users', @HandleCreateUser);
  FServer.Get('/api/users/*', @HandleGetUser);
end;

destructor TAPIServer.Destroy;
begin
  FServer.Free;
  FUsers.Free;
  inherited Destroy;
end;

procedure TAPIServer.HandleHealth(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
begin
  AResponse := TSimpleHTTPServer.OKResponse(
    '{"status": "healthy", "version": "1.0.0", "time": "' + 
    DateTimeToStr(Now) + '"}'
  );
end;

procedure TAPIServer.HandleGetUsers(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
var
  JSON: string;
  i: Integer;
  Parts: TStringList;
begin
  JSON := '[';
  Parts := TStringList.Create;
  try
    for i := 0 to FUsers.Count - 1 do
    begin
      Parts.Delimiter := ':';
      Parts.DelimitedText := FUsers[i];
      
      if Parts.Count >= 3 then
      begin
        if i > 0 then JSON := JSON + ',';
        JSON := JSON + Format('{"id":%s,"name":"%s","email":"%s"}',
          [Parts[0], Parts[1], Parts[2]]);
      end;
    end;
  finally
    Parts.Free;
  end;
  JSON := JSON + ']';
  
  AResponse := TSimpleHTTPServer.OKResponse(JSON);
end;

procedure TAPIServer.HandleCreateUser(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
var
  NewID: Integer;
begin
  NewID := FUsers.Count + 1;
  
  // Parse body (simplified)
  FUsers.Add(Format('%d:New User:new@email.com', [NewID]));
  
  AResponse.StatusCode := 201;
  AResponse.Body := Format('{"id": %d, "message": "User created"}', [NewID]);
  AResponse.ContentType := 'application/json';
end;

procedure TAPIServer.HandleGetUser(const ARequest: THTTPRequest; var AResponse: THTTPResponse);
var
  IDStr: string;
  i: Integer;
  Parts: TStringList;
begin
  // Extract ID from path /api/users/1
  IDStr := Copy(ARequest.Path, Length('/api/users/') + 1, MaxInt);
  
  Parts := TStringList.Create;
  try
    for i := 0 to FUsers.Count - 1 do
    begin
      Parts.Delimiter := ':';
      Parts.DelimitedText := FUsers[i];
      
      if (Parts.Count >= 3) and (Parts[0] = IDStr) then
      begin
        AResponse := TSimpleHTTPServer.OKResponse(
          Format('{"id":%s,"name":"%s","email":"%s"}',
            [Parts[0], Parts[1], Parts[2]])
        );
        Exit;
      end;
    end;
  finally
    Parts.Free;
  end;
  
  AResponse := TSimpleHTTPServer.ErrorResponse(404, 'User not found');
end;

procedure TAPIServer.Run;
begin
  WriteLn('Starting API Server...');
  FServer.Start;
end;

var
  App: TAPIServer;
  Port: Integer;
begin
  Port := StrToIntDef(GetEnvironmentVariable('PORT'), 8080);
  
  WriteLn('=== Pascal API Server ===');
  WriteLn('Port: ', Port);
  
  App := TAPIServer.Create(Port);
  try
    App.Run;
  finally
    App.Free;
  end;
end.
```

---

## 71.4 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Pascal API Service
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=secret
      - LOG_LEVEL=info
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # PostgreSQL Database
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

---

## 71.5 Makefile สำหรับ Docker

```makefile
# Makefile
APP_NAME = myapp
VERSION = 1.0.0
DOCKER_REGISTRY = registry.example.com
IMAGE_NAME = $(DOCKER_REGISTRY)/$(APP_NAME):$(VERSION)

.PHONY: build test run stop clean deploy

# Build Docker image
build:
	docker build -t $(APP_NAME):$(VERSION) .
	docker tag $(APP_NAME):$(VERSION) $(APP_NAME):latest

# Build and test
test:
	docker build --target builder -t $(APP_NAME)-test .
	docker run --rm $(APP_NAME)-test /app/run_tests

# Run with docker-compose
run:
	docker-compose up -d
	@echo "Server started at http://localhost:8080"

# Stop containers
stop:
	docker-compose down

# Clean images
clean:
	docker-compose down -v
	docker rmi $(APP_NAME):$(VERSION) $(APP_NAME):latest 2>/dev/null || true

# Push to registry
push: build
	docker tag $(APP_NAME):$(VERSION) $(IMAGE_NAME)
	docker push $(IMAGE_NAME)

# Deploy to production
deploy: push
	ssh user@production "docker pull $(IMAGE_NAME) && docker-compose up -d"

# Logs
logs:
	docker-compose logs -f api

# Shell in container
shell:
	docker-compose exec api /bin/sh

# Database migration
migrate:
	docker-compose exec api /app/migrate
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Dockerfile** - สร้าง multi-stage build สำหรับ Pascal
2. **HTTP Server** - สร้าง web server ใน Pascal
3. **REST API** - สร้าง API endpoints
4. **Docker Compose** - ประกอบ services หลายตัว
5. **Makefile** - Automate build และ deployment

Docker ช่วยให้แอปพลิเคชัน Pascal สามารถ deploy ได้ง่ายและ consistent ทุกที่ ไม่ว่าจะเป็น development, staging, หรือ production
