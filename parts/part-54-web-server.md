# Part 54 - Web Server ด้วย Lazarus/Pascal

## บทนำ

Lazarus มี Framework หลายตัวสำหรับสร้าง Web Server:
- **fpWeb** - Built-in Web Framework ของ Free Pascal
- **Brook Framework** - Framework ที่ใช้ libbrook
- **Indy** - TCP/IP Library ที่รองรับ HTTP

---

## fpWeb Framework

### การติดตั้ง

เพิ่มใน .lpr:
```pascal
uses
  ..., fphttpapp, fpweb, httpdefs, httproute;
```

### Web Application พื้นฐาน

```pascal
program BasicWebServer;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, fphttpapp, fpweb, httpdefs;

type
  TMainModule = class(TFPWebModule)
  published
    procedure HandleHome(Sender: TObject; ARequest: TRequest; 
                         AResponse: TResponse; var Handled: Boolean);
    procedure HandleAbout(Sender: TObject; ARequest: TRequest;
                          AResponse: TResponse; var Handled: Boolean);
    procedure HandleContact(Sender: TObject; ARequest: TRequest;
                            AResponse: TResponse; var Handled: Boolean);
  end;

procedure TMainModule.HandleHome(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
begin
  AResponse.ContentType := 'text/html; charset=utf-8';
  AResponse.Content :=
    '<!DOCTYPE html>' +
    '<html lang="th">' +
    '<head><meta charset="UTF-8"><title>หน้าแรก</title>' +
    '<style>body{font-family:sans-serif;margin:40px;}</style></head>' +
    '<body>' +
    '<h1>ยินดีต้อนรับสู่ Web Server ด้วย Lazarus!</h1>' +
    '<nav>' +
    '<a href="/">หน้าแรก</a> | ' +
    '<a href="/about">เกี่ยวกับ</a> | ' +
    '<a href="/contact">ติดต่อ</a>' +
    '</nav>' +
    '<p>เวลาปัจจุบัน: ' + FormatDateTime('dd/mm/yyyy hh:nn:ss', Now) + '</p>' +
    '</body></html>';
  Handled := True;
end;

procedure TMainModule.HandleAbout(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
begin
  AResponse.ContentType := 'text/html; charset=utf-8';
  AResponse.Content :=
    '<!DOCTYPE html><html><head><meta charset="UTF-8"><title>เกี่ยวกับ</title></head>' +
    '<body><h1>เกี่ยวกับเรา</h1>' +
    '<p>นี่คือตัวอย่าง Web Server ที่สร้างด้วย Lazarus Pascal</p>' +
    '<a href="/">กลับหน้าแรก</a></body></html>';
  Handled := True;
end;

procedure TMainModule.HandleContact(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
begin
  AResponse.ContentType := 'text/html; charset=utf-8';
  AResponse.Content :=
    '<!DOCTYPE html><html><head><meta charset="UTF-8"><title>ติดต่อ</title></head>' +
    '<body><h1>ติดต่อเรา</h1>' +
    '<p>Email: contact@example.com</p>' +
    '<p>โทร: 02-123-4567</p>' +
    '<a href="/">กลับหน้าแรก</a></body></html>';
  Handled := True;
end;

initialization
  RegisterHTTPModule('main', TMainModule);

var
  App: TFPHTTPApplication;
begin
  App := TFPHTTPApplication.Create(nil);
  try
    App.Threaded := False;
    App.Port := 9000;
    WriteLn('Web Server เริ่มทำงานที่ http://localhost:9000');
    App.Run;
  finally
    App.Free;
  end;
end.
```

---

## HTTP Router

```pascal
unit WebRouter;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, httpdefs, Generics.Collections;

type
  THTTPMethod = (hmGET, hmPOST, hmPUT, hmPATCH, hmDELETE, hmOPTIONS, hmANY);
  
  TRouteHandler = procedure(ARequest: TRequest; AResponse: TResponse) of object;
  
  TMiddlewareProc = function(ARequest: TRequest; AResponse: TResponse): Boolean of object;

  TRouteParam = record
    Name: string;
    Value: string;
  end;
  
  TRoute = class
  private
    FMethod: THTTPMethod;
    FPattern: string;
    FHandler: TRouteHandler;
    FMiddlewares: specialize TList<TMiddlewareProc>;
    FParamNames: TStringList;
  public
    constructor Create(AMethod: THTTPMethod; const APattern: string; AHandler: TRouteHandler);
    destructor Destroy; override;
    function Matches(AMethod: THTTPMethod; const APath: string; 
                     out AParams: array of TRouteParam): Boolean;
    procedure AddMiddleware(AMiddleware: TMiddlewareProc);
    property Method: THTTPMethod read FMethod;
    property Pattern: string read FPattern;
    property Handler: TRouteHandler read FHandler;
    property Middlewares: specialize TList<TMiddlewareProc> read FMiddlewares;
  end;

  TRouteGroup = class
  private
    FRouter: TObject; { TRouter }
    FPrefix: string;
    FMiddlewares: specialize TList<TMiddlewareProc>;
  public
    constructor Create(ARouter: TObject; const APrefix: string);
    destructor Destroy; override;
    function Get(const APath: string; AHandler: TRouteHandler): TRoute;
    function Post(const APath: string; AHandler: TRouteHandler): TRoute;
    function Put(const APath: string; AHandler: TRouteHandler): TRoute;
    function Delete(const APath: string; AHandler: TRouteHandler): TRoute;
    function Use(AMiddleware: TMiddlewareProc): TRouteGroup;
  end;

  TRouter = class
  private
    FRoutes: specialize TObjectList<TRoute>;
    FMiddlewares: specialize TList<TMiddlewareProc>;
    FNotFoundHandler: TRouteHandler;
    FErrorHandler: procedure(ARequest: TRequest; AResponse: TResponse; 
                              const AError: string) of object;
    function MethodToEnum(const AMethod: string): THTTPMethod;
  public
    constructor Create;
    destructor Destroy; override;
    
    function Get(const APath: string; AHandler: TRouteHandler): TRoute;
    function Post(const APath: string; AHandler: TRouteHandler): TRoute;
    function Put(const APath: string; AHandler: TRouteHandler): TRoute;
    function Patch(const APath: string; AHandler: TRouteHandler): TRoute;
    function Delete(const APath: string; AHandler: TRouteHandler): TRoute;
    function Group(const APrefix: string): TRouteGroup;
    procedure Use(AMiddleware: TMiddlewareProc);
    
    procedure HandleRequest(ARequest: TRequest; AResponse: TResponse);
    
    property NotFoundHandler: TRouteHandler read FNotFoundHandler write FNotFoundHandler;
  end;

implementation

{ TRoute }
constructor TRoute.Create(AMethod: THTTPMethod; const APattern: string; AHandler: TRouteHandler);
var
  Parts: TStringArray;
  Part: string;
begin
  FMethod := AMethod;
  FPattern := APattern;
  FHandler := AHandler;
  FMiddlewares := specialize TList<TMiddlewareProc>.Create;
  FParamNames := TStringList.Create;
  
  { Extract parameter names from pattern like /users/:id/posts/:postId }
  Parts := APattern.Split(['/']);
  for Part in Parts do
    if (Length(Part) > 0) and (Part[1] = ':') then
      FParamNames.Add(Copy(Part, 2, MaxInt));
end;

destructor TRoute.Destroy;
begin
  FMiddlewares.Free;
  FParamNames.Free;
  inherited;
end;

function TRoute.Matches(AMethod: THTTPMethod; const APath: string;
  out AParams: array of TRouteParam): Boolean;
var
  PatternParts, PathParts: TStringArray;
  I, ParamIdx: Integer;
begin
  Result := False;
  
  if (FMethod <> hmANY) and (FMethod <> AMethod) then Exit;
  
  PatternParts := FPattern.Split(['/']);
  PathParts := APath.Split(['/']);
  
  if Length(PatternParts) <> Length(PathParts) then Exit;
  
  ParamIdx := 0;
  for I := 0 to High(PatternParts) do
  begin
    if (Length(PatternParts[I]) > 0) and (PatternParts[I][1] = ':') then
    begin
      { Parameter }
      if ParamIdx <= High(AParams) then
      begin
        AParams[ParamIdx].Name := Copy(PatternParts[I], 2, MaxInt);
        AParams[ParamIdx].Value := PathParts[I];
        Inc(ParamIdx);
      end;
    end
    else if PatternParts[I] <> PathParts[I] then
      Exit;
  end;
  
  Result := True;
end;

procedure TRoute.AddMiddleware(AMiddleware: TMiddlewareProc);
begin
  FMiddlewares.Add(AMiddleware);
end;

{ TRouter }
constructor TRouter.Create;
begin
  FRoutes := specialize TObjectList<TRoute>.Create(True);
  FMiddlewares := specialize TList<TMiddlewareProc>.Create;
end;

destructor TRouter.Destroy;
begin
  FRoutes.Free;
  FMiddlewares.Free;
  inherited;
end;

function TRouter.MethodToEnum(const AMethod: string): THTTPMethod;
var
  M: string;
begin
  M := UpperCase(AMethod);
  if M = 'GET' then Result := hmGET
  else if M = 'POST' then Result := hmPOST
  else if M = 'PUT' then Result := hmPUT
  else if M = 'PATCH' then Result := hmPATCH
  else if M = 'DELETE' then Result := hmDELETE
  else if M = 'OPTIONS' then Result := hmOPTIONS
  else Result := hmANY;
end;

function TRouter.Get(const APath: string; AHandler: TRouteHandler): TRoute;
begin
  Result := TRoute.Create(hmGET, APath, AHandler);
  FRoutes.Add(Result);
end;

function TRouter.Post(const APath: string; AHandler: TRouteHandler): TRoute;
begin
  Result := TRoute.Create(hmPOST, APath, AHandler);
  FRoutes.Add(Result);
end;

function TRouter.Put(const APath: string; AHandler: TRouteHandler): TRoute;
begin
  Result := TRoute.Create(hmPUT, APath, AHandler);
  FRoutes.Add(Result);
end;

function TRouter.Patch(const APath: string; AHandler: TRouteHandler): TRoute;
begin
  Result := TRoute.Create(hmPATCH, APath, AHandler);
  FRoutes.Add(Result);
end;

function TRouter.Delete(const APath: string; AHandler: TRouteHandler): TRoute;
begin
  Result := TRoute.Create(hmDELETE, APath, AHandler);
  FRoutes.Add(Result);
end;

function TRouter.Group(const APrefix: string): TRouteGroup;
begin
  Result := TRouteGroup.Create(Self, APrefix);
end;

procedure TRouter.Use(AMiddleware: TMiddlewareProc);
begin
  FMiddlewares.Add(AMiddleware);
end;

procedure TRouter.HandleRequest(ARequest: TRequest; AResponse: TResponse);
var
  Route: TRoute;
  Method: THTTPMethod;
  Params: array[0..9] of TRouteParam;
  Middleware: TMiddlewareProc;
  I: Integer;
begin
  Method := MethodToEnum(ARequest.Method);
  
  { Run global middlewares }
  for I := 0 to FMiddlewares.Count - 1 do
  begin
    Middleware := FMiddlewares[I];
    if not Middleware(ARequest, AResponse) then
    begin
      { Middleware หยุดการทำงาน }
      AResponse.SendResponse;
      Exit;
    end;
  end;
  
  { Find matching route }
  for Route in FRoutes do
  begin
    if Route.Matches(Method, ARequest.PathInfo, Params) then
    begin
      { Run route middlewares }
      for I := 0 to Route.Middlewares.Count - 1 do
      begin
        Middleware := Route.Middlewares[I];
        if not Middleware(ARequest, AResponse) then
        begin
          AResponse.SendResponse;
          Exit;
        end;
      end;
      
      { Execute handler }
      try
        Route.Handler(ARequest, AResponse);
      except
        on E: Exception do
        begin
          AResponse.Code := 500;
          AResponse.ContentType := 'application/json';
          AResponse.Content := '{"error":"' + E.Message + '"}';
        end;
      end;
      
      AResponse.SendResponse;
      Exit;
    end;
  end;
  
  { Not Found }
  if Assigned(FNotFoundHandler) then
    FNotFoundHandler(ARequest, AResponse)
  else
  begin
    AResponse.Code := 404;
    AResponse.ContentType := 'application/json';
    AResponse.Content := '{"error":"ไม่พบเส้นทางที่ขอ"}';
  end;
  
  AResponse.SendResponse;
end;

end.
```

---

## Session Management

```pascal
unit SessionManager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, httpdefs, DateUtils, SyncObjs,
  Generics.Collections;

type
  TSessionData = class
  private
    FData: TStringList;
    FCreatedAt: TDateTime;
    FLastAccessAt: TDateTime;
    FSessionID: string;
    FUserID: Integer;
  public
    constructor Create(const ASessionID: string);
    destructor Destroy; override;
    procedure SetValue(const AKey, AValue: string);
    function GetValue(const AKey: string; const ADefault: string = ''): string;
    procedure Remove(const AKey: string);
    procedure Clear;
    function IsExpired(ATimeoutMinutes: Integer): Boolean;
    procedure Touch;
    
    property SessionID: string read FSessionID;
    property UserID: Integer read FUserID write FUserID;
    property CreatedAt: TDateTime read FCreatedAt;
    property LastAccessAt: TDateTime read FLastAccessAt;
  end;

  TSessionManager = class
  private
    FSessions: specialize TObjectDictionary<string, TSessionData>;
    FLock: TCriticalSection;
    FTimeoutMinutes: Integer;
    FCookieName: string;
    FSecure: Boolean;
    FHTTPOnly: Boolean;
    
    function GenerateSessionID: string;
    procedure CleanExpiredSessions;
  public
    constructor Create;
    destructor Destroy; override;
    
    function GetOrCreateSession(ARequest: TRequest; AResponse: TResponse): TSessionData;
    function GetSession(const ASessionID: string): TSessionData;
    procedure DestroySession(const ASessionID: string; AResponse: TResponse);
    procedure DestroyAllUserSessions(AUserID: Integer);
    function SessionCount: Integer;
    
    property TimeoutMinutes: Integer read FTimeoutMinutes write FTimeoutMinutes;
    property CookieName: string read FCookieName write FCookieName;
    property Secure: Boolean read FSecure write FSecure;
    property HTTPOnly: Boolean read FHTTPOnly write FHTTPOnly;
  end;

var
  SessionMgr: TSessionManager;

implementation

{ TSessionData }
constructor TSessionData.Create(const ASessionID: string);
begin
  FSessionID := ASessionID;
  FData := TStringList.Create;
  FCreatedAt := Now;
  FLastAccessAt := Now;
  FUserID := 0;
end;

destructor TSessionData.Destroy;
begin
  FData.Free;
  inherited;
end;

procedure TSessionData.SetValue(const AKey, AValue: string);
begin
  FData.Values[AKey] := AValue;
  Touch;
end;

function TSessionData.GetValue(const AKey: string; const ADefault: string): string;
begin
  Result := FData.Values[AKey];
  if Result = '' then Result := ADefault;
  Touch;
end;

procedure TSessionData.Remove(const AKey: string);
var
  Idx: Integer;
begin
  Idx := FData.IndexOfName(AKey);
  if Idx >= 0 then FData.Delete(Idx);
end;

procedure TSessionData.Clear;
begin
  FData.Clear;
end;

function TSessionData.IsExpired(ATimeoutMinutes: Integer): Boolean;
begin
  Result := MinutesBetween(Now, FLastAccessAt) > ATimeoutMinutes;
end;

procedure TSessionData.Touch;
begin
  FLastAccessAt := Now;
end;

{ TSessionManager }
constructor TSessionManager.Create;
begin
  FSessions := specialize TObjectDictionary<string, TSessionData>.Create([doOwnsValues]);
  FLock := TCriticalSection.Create;
  FTimeoutMinutes := 30;
  FCookieName := 'SESSIONID';
  FSecure := False;
  FHTTPOnly := True;
end;

destructor TSessionManager.Destroy;
begin
  FSessions.Free;
  FLock.Free;
  inherited;
end;

function TSessionManager.GenerateSessionID: string;
var
  I: Integer;
  Chars: string;
begin
  Chars := 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  Result := '';
  for I := 1 to 32 do
    Result := Result + Chars[Random(Length(Chars)) + 1];
  Result := Result + IntToStr(DateTimeToUnix(Now));
end;

procedure TSessionManager.CleanExpiredSessions;
var
  SessionID: string;
  ToDelete: TStringList;
begin
  ToDelete := TStringList.Create;
  try
    FLock.Enter;
    try
      for SessionID in FSessions.Keys do
        if FSessions[SessionID].IsExpired(FTimeoutMinutes) then
          ToDelete.Add(SessionID);
      
      for SessionID in ToDelete do
        FSessions.Remove(SessionID);
    finally
      FLock.Leave;
    end;
  finally
    ToDelete.Free;
  end;
end;

function TSessionManager.GetOrCreateSession(ARequest: TRequest; AResponse: TResponse): TSessionData;
var
  SessionID: string;
  Cookie: TCookie;
begin
  { ดึง Session ID จาก Cookie }
  SessionID := ARequest.CookieFields.Values[FCookieName];
  
  FLock.Enter;
  try
    if (SessionID <> '') and FSessions.ContainsKey(SessionID) then
    begin
      Result := FSessions[SessionID];
      if Result.IsExpired(FTimeoutMinutes) then
      begin
        FSessions.Remove(SessionID);
        Result := nil;
      end;
    end
    else
      Result := nil;
    
    if Result = nil then
    begin
      { สร้าง Session ใหม่ }
      SessionID := GenerateSessionID;
      Result := TSessionData.Create(SessionID);
      FSessions.Add(SessionID, Result);
      
      { ตั้ง Cookie }
      Cookie := AResponse.Cookies.Add;
      Cookie.Name := FCookieName;
      Cookie.Value := SessionID;
      Cookie.Path := '/';
      Cookie.HttpOnly := FHTTPOnly;
      Cookie.Secure := FSecure;
      Cookie.MaxAge := FTimeoutMinutes * 60;
    end;
    
    Result.Touch;
  finally
    FLock.Leave;
  end;
  
  { ทำความสะอาด Session ที่หมดอายุทุก 100 ครั้ง }
  if Random(100) = 0 then
    CleanExpiredSessions;
end;

function TSessionManager.GetSession(const ASessionID: string): TSessionData;
begin
  FLock.Enter;
  try
    if FSessions.ContainsKey(ASessionID) then
      Result := FSessions[ASessionID]
    else
      Result := nil;
  finally
    FLock.Leave;
  end;
end;

procedure TSessionManager.DestroySession(const ASessionID: string; AResponse: TResponse);
var
  Cookie: TCookie;
begin
  FLock.Enter;
  try
    FSessions.Remove(ASessionID);
  finally
    FLock.Leave;
  end;
  
  { ลบ Cookie }
  Cookie := AResponse.Cookies.Add;
  Cookie.Name := FCookieName;
  Cookie.Value := '';
  Cookie.MaxAge := -1;
end;

procedure TSessionManager.DestroyAllUserSessions(AUserID: Integer);
var
  SessionID: string;
  ToDelete: TStringList;
begin
  ToDelete := TStringList.Create;
  try
    FLock.Enter;
    try
      for SessionID in FSessions.Keys do
        if FSessions[SessionID].UserID = AUserID then
          ToDelete.Add(SessionID);
      
      for SessionID in ToDelete do
        FSessions.Remove(SessionID);
    finally
      FLock.Leave;
    end;
  finally
    ToDelete.Free;
  end;
end;

function TSessionManager.SessionCount: Integer;
begin
  FLock.Enter;
  try
    Result := FSessions.Count;
  finally
    FLock.Leave;
  end;
end;

initialization
  SessionMgr := TSessionManager.Create;
  Randomize;

finalization
  SessionMgr.Free;

end.
```

---

## Template Engine (Simple)

```pascal
unit SimpleTemplate;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TTemplateVars = class
  private
    FVars: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure SetVar(const AName, AValue: string);
    function GetVar(const AName: string): string;
    procedure SetIntVar(const AName: string; AValue: Integer);
    procedure SetBoolVar(const AName: string; AValue: Boolean);
    procedure SetDateVar(const AName: string; AValue: TDateTime; const AFormat: string = 'dd/mm/yyyy');
    procedure Clear;
  end;

  TTemplateEngine = class
  private
    FTemplateDir: string;
    FCache: TStringList;
    procedure ProcessIfBlocks(var AContent: string; AVars: TTemplateVars);
    procedure ProcessForBlocks(var AContent: string; AVars: TTemplateVars);
    function ReplaceVars(const AContent: string; AVars: TTemplateVars): string;
    function LoadTemplate(const AFileName: string): string;
  public
    constructor Create(const ATemplateDir: string);
    destructor Destroy; override;
    function Render(const ATemplateName: string; AVars: TTemplateVars): string;
    function RenderString(const ATemplate: string; AVars: TTemplateVars): string;
    procedure EnableCache(AEnable: Boolean);
  end;

implementation

{ TTemplateVars }
constructor TTemplateVars.Create;
begin
  FVars := TStringList.Create;
end;

destructor TTemplateVars.Destroy;
begin
  FVars.Free;
  inherited;
end;

procedure TTemplateVars.SetVar(const AName, AValue: string);
begin
  FVars.Values[AName] := AValue;
end;

function TTemplateVars.GetVar(const AName: string): string;
begin
  Result := FVars.Values[AName];
end;

procedure TTemplateVars.SetIntVar(const AName: string; AValue: Integer);
begin
  FVars.Values[AName] := IntToStr(AValue);
end;

procedure TTemplateVars.SetBoolVar(const AName: string; AValue: Boolean);
begin
  FVars.Values[AName] := BoolToStr(AValue, 'true', 'false');
end;

procedure TTemplateVars.SetDateVar(const AName: string; AValue: TDateTime; const AFormat: string);
begin
  FVars.Values[AName] := FormatDateTime(AFormat, AValue);
end;

procedure TTemplateVars.Clear;
begin
  FVars.Clear;
end;

{ TTemplateEngine }
constructor TTemplateEngine.Create(const ATemplateDir: string);
begin
  FTemplateDir := ATemplateDir;
  FCache := TStringList.Create;
end;

destructor TTemplateEngine.Destroy;
begin
  FCache.Free;
  inherited;
end;

function TTemplateEngine.LoadTemplate(const AFileName: string): string;
var
  FullPath: string;
  Lines: TStringList;
  CacheIdx: Integer;
begin
  FullPath := IncludeTrailingPathDelimiter(FTemplateDir) + AFileName;
  
  { Check Cache }
  CacheIdx := FCache.IndexOfName(FullPath);
  if CacheIdx >= 0 then
  begin
    Result := FCache.ValueFromIndex[CacheIdx];
    Exit;
  end;
  
  if not FileExists(FullPath) then
    raise Exception.CreateFmt('ไม่พบ Template: %s', [FullPath]);
  
  Lines := TStringList.Create;
  try
    Lines.LoadFromFile(FullPath, TEncoding.UTF8);
    Result := Lines.Text;
    FCache.Values[FullPath] := Result;
  finally
    Lines.Free;
  end;
end;

function TTemplateEngine.ReplaceVars(const AContent: string; AVars: TTemplateVars): string;
var
  I: Integer;
  VarName, VarValue: string;
begin
  Result := AContent;
  for I := 0 to AVars.FVars.Count - 1 do
  begin
    VarName := '{{ ' + AVars.FVars.Names[I] + ' }}';
    VarValue := AVars.FVars.ValueFromIndex[I];
    Result := StringReplace(Result, VarName, VarValue, [rfReplaceAll]);
    { Also handle without spaces }
    VarName := '{{' + AVars.FVars.Names[I] + '}}';
    Result := StringReplace(Result, VarName, VarValue, [rfReplaceAll]);
  end;
end;

procedure TTemplateEngine.ProcessIfBlocks(var AContent: string; AVars: TTemplateVars);
var
  StartTag, EndTag: string;
  StartPos, EndPos, CondStart, CondEnd: Integer;
  Condition, VarName, Block: string;
  VarValue: string;
  Negate: Boolean;
begin
  { Process {% if varname %} ... {% endif %} }
  while True do
  begin
    StartPos := Pos('{% if ', AContent);
    if StartPos = 0 then Break;
    
    CondStart := StartPos + 6;
    CondEnd := PosEx(' %}', AContent, CondStart);
    if CondEnd = 0 then Break;
    
    Condition := Trim(Copy(AContent, CondStart, CondEnd - CondStart));
    
    Negate := False;
    if Copy(Condition, 1, 4) = 'not ' then
    begin
      Negate := True;
      VarName := Trim(Copy(Condition, 5, MaxInt));
    end
    else
      VarName := Condition;
    
    EndTag := '{% endif %}';
    EndPos := PosEx(EndTag, AContent, CondEnd);
    if EndPos = 0 then Break;
    
    Block := Copy(AContent, CondEnd + 3, EndPos - CondEnd - 3);
    
    VarValue := AVars.GetVar(VarName);
    
    if (not Negate and (VarValue <> '') and (LowerCase(VarValue) <> 'false')) or
       (Negate and ((VarValue = '') or (LowerCase(VarValue) = 'false'))) then
      { แสดง Block }
      AContent := Copy(AContent, 1, StartPos - 1) + Block + 
                  Copy(AContent, EndPos + Length(EndTag), MaxInt)
    else
      { ซ่อน Block }
      AContent := Copy(AContent, 1, StartPos - 1) + 
                  Copy(AContent, EndPos + Length(EndTag), MaxInt);
  end;
end;

procedure TTemplateEngine.ProcessForBlocks(var AContent: string; AVars: TTemplateVars);
begin
  { TODO: Implement for loops }
  { {% for item in items %} ... {% endfor %} }
end;

function TTemplateEngine.Render(const ATemplateName: string; AVars: TTemplateVars): string;
var
  Template: string;
begin
  Template := LoadTemplate(ATemplateName);
  Result := RenderString(Template, AVars);
end;

function TTemplateEngine.RenderString(const ATemplate: string; AVars: TTemplateVars): string;
begin
  Result := ATemplate;
  ProcessIfBlocks(Result, AVars);
  ProcessForBlocks(Result, AVars);
  Result := ReplaceVars(Result, AVars);
end;

procedure TTemplateEngine.EnableCache(AEnable: Boolean);
begin
  if not AEnable then
    FCache.Clear;
end;

end.
```

---

## ตัวอย่าง Simple Web Server สมบูรณ์

```pascal
program SimpleWebServer;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, fphttpapp, fpweb, httpdefs,
  SessionManager, SimpleTemplate;

type
  TWebApp = class(TFPWebModule)
  private
    FTemplates: TTemplateEngine;
    
    procedure ServeStaticFile(const APath: string; AResponse: TResponse);
    procedure SendError(AResponse: TResponse; ACode: Integer; const AMessage: string);
    function GetUserFromSession(ARequest: TRequest): Integer;
  published
    procedure HandleHome(Sender: TObject; ARequest: TRequest;
                         AResponse: TResponse; var Handled: Boolean);
    procedure HandleLogin(Sender: TObject; ARequest: TRequest;
                          AResponse: TResponse; var Handled: Boolean);
    procedure HandleLogout(Sender: TObject; ARequest: TRequest;
                           AResponse: TResponse; var Handled: Boolean);
    procedure HandleDashboard(Sender: TObject; ARequest: TRequest;
                              AResponse: TResponse; var Handled: Boolean);
    procedure HandleStatic(Sender: TObject; ARequest: TRequest;
                           AResponse: TResponse; var Handled: Boolean);
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
  end;

constructor TWebApp.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FTemplates := TTemplateEngine.Create('templates');
end;

destructor TWebApp.Destroy;
begin
  FTemplates.Free;
  inherited;
end;

procedure TWebApp.ServeStaticFile(const APath: string; AResponse: TResponse);
var
  Content: TFileStream;
  Ext: string;
begin
  if not FileExists(APath) then
  begin
    AResponse.Code := 404;
    AResponse.Content := 'ไม่พบไฟล์';
    Exit;
  end;
  
  Ext := LowerCase(ExtractFileExt(APath));
  case Ext of
    '.html': AResponse.ContentType := 'text/html; charset=utf-8';
    '.css':  AResponse.ContentType := 'text/css';
    '.js':   AResponse.ContentType := 'application/javascript';
    '.png':  AResponse.ContentType := 'image/png';
    '.jpg':  AResponse.ContentType := 'image/jpeg';
    '.gif':  AResponse.ContentType := 'image/gif';
    '.ico':  AResponse.ContentType := 'image/x-icon';
  else
    AResponse.ContentType := 'application/octet-stream';
  end;
  
  Content := TFileStream.Create(APath, fmOpenRead or fmShareDenyWrite);
  try
    AResponse.ContentStream := Content;
    AResponse.FreeContentStream := True;
  except
    Content.Free;
    raise;
  end;
end;

procedure TWebApp.SendError(AResponse: TResponse; ACode: Integer; const AMessage: string);
begin
  AResponse.Code := ACode;
  AResponse.ContentType := 'text/html; charset=utf-8';
  AResponse.Content := Format(
    '<!DOCTYPE html><html><body><h1>ข้อผิดพลาด %d</h1><p>%s</p></body></html>',
    [ACode, AMessage]);
end;

function TWebApp.GetUserFromSession(ARequest: TRequest): Integer;
var
  Session: TSessionData;
  SessionID: string;
begin
  Result := 0;
  SessionID := ARequest.CookieFields.Values['SESSIONID'];
  if SessionID = '' then Exit;
  
  Session := SessionMgr.GetSession(SessionID);
  if Session <> nil then
    Result := Session.UserID;
end;

procedure TWebApp.HandleHome(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
var
  Vars: TTemplateVars;
  UserID: Integer;
begin
  UserID := GetUserFromSession(ARequest);
  
  Vars := TTemplateVars.Create;
  try
    Vars.SetVar('title', 'หน้าแรก');
    Vars.SetVar('year', FormatDateTime('yyyy', Now));
    Vars.SetBoolVar('is_logged_in', UserID > 0);
    
    if UserID > 0 then
      Vars.SetVar('username', 'ผู้ใช้ #' + IntToStr(UserID));
    
    AResponse.ContentType := 'text/html; charset=utf-8';
    AResponse.Content := FTemplates.Render('home.html', Vars);
    Handled := True;
  finally
    Vars.Free;
  end;
end;

procedure TWebApp.HandleLogin(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
var
  Session: TSessionData;
  Username, Password: string;
  Vars: TTemplateVars;
begin
  Handled := True;
  
  if ARequest.Method = 'POST' then
  begin
    Username := ARequest.ContentFields.Values['username'];
    Password := ARequest.ContentFields.Values['password'];
    
    { ตรวจสอบ Credentials (ตัวอย่าง) }
    if (Username = 'admin') and (Password = 'password') then
    begin
      Session := SessionMgr.GetOrCreateSession(ARequest, AResponse);
      Session.UserID := 1;
      Session.SetValue('username', Username);
      Session.SetValue('role', 'admin');
      
      AResponse.Code := 302;
      AResponse.SetFieldByName('Location', '/dashboard');
      Exit;
    end
    else
    begin
      Vars := TTemplateVars.Create;
      try
        Vars.SetVar('title', 'เข้าสู่ระบบ');
        Vars.SetVar('error', 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง');
        Vars.SetVar('username', Username);
        AResponse.Content := FTemplates.Render('login.html', Vars);
      finally
        Vars.Free;
      end;
      Exit;
    end;
  end;
  
  { แสดงฟอร์ม Login }
  Vars := TTemplateVars.Create;
  try
    Vars.SetVar('title', 'เข้าสู่ระบบ');
    AResponse.ContentType := 'text/html; charset=utf-8';
    AResponse.Content := FTemplates.Render('login.html', Vars);
  finally
    Vars.Free;
  end;
end;

procedure TWebApp.HandleLogout(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
var
  SessionID: string;
begin
  SessionID := ARequest.CookieFields.Values['SESSIONID'];
  if SessionID <> '' then
    SessionMgr.DestroySession(SessionID, AResponse);
  
  AResponse.Code := 302;
  AResponse.SetFieldByName('Location', '/');
  Handled := True;
end;

procedure TWebApp.HandleDashboard(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
var
  UserID: Integer;
  Vars: TTemplateVars;
  Session: TSessionData;
  SessionID: string;
begin
  UserID := GetUserFromSession(ARequest);
  
  if UserID = 0 then
  begin
    AResponse.Code := 302;
    AResponse.SetFieldByName('Location', '/login');
    Handled := True;
    Exit;
  end;
  
  SessionID := ARequest.CookieFields.Values['SESSIONID'];
  Session := SessionMgr.GetSession(SessionID);
  
  Vars := TTemplateVars.Create;
  try
    Vars.SetVar('title', 'แดชบอร์ด');
    Vars.SetVar('username', Session.GetValue('username'));
    Vars.SetVar('role', Session.GetValue('role'));
    Vars.SetVar('user_id', IntToStr(UserID));
    Vars.SetVar('session_count', IntToStr(SessionMgr.SessionCount));
    Vars.SetVar('current_time', FormatDateTime('dd/mm/yyyy hh:nn:ss', Now));
    
    AResponse.ContentType := 'text/html; charset=utf-8';
    AResponse.Content := FTemplates.Render('dashboard.html', Vars);
    Handled := True;
  finally
    Vars.Free;
  end;
end;

procedure TWebApp.HandleStatic(Sender: TObject; ARequest: TRequest;
  AResponse: TResponse; var Handled: Boolean);
var
  FilePath: string;
begin
  { /static/css/style.css -> static/css/style.css }
  FilePath := Copy(ARequest.PathInfo, 2, MaxInt);
  FilePath := StringReplace(FilePath, '/', PathDelim, [rfReplaceAll]);
  
  { ป้องกัน Path Traversal }
  if Pos('..', FilePath) > 0 then
  begin
    SendError(AResponse, 403, 'Forbidden');
    Handled := True;
    Exit;
  end;
  
  ServeStaticFile(FilePath, AResponse);
  Handled := True;
end;

initialization
  RegisterHTTPModule('webapp', TWebApp);

var
  App: TFPHTTPApplication;
begin
  App := TFPHTTPApplication.Create(nil);
  try
    App.Threaded := True;
    App.Port := 8080;
    WriteLn('Web Server เริ่มทำงานที่ http://localhost:8080');
    WriteLn('กด Ctrl+C เพื่อหยุดเซิร์ฟเวอร์');
    App.Run;
  finally
    App.Free;
  end;
end.
```

---

## Template ตัวอย่าง

**templates/home.html:**
```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ title }}</title>
  <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
  <nav class="navbar">
    <div class="container">
      <a href="/" class="brand">MyApp</a>
      <div class="nav-links">
        {% if is_logged_in %}
        <span>สวัสดี, {{ username }}</span>
        <a href="/dashboard">แดชบอร์ด</a>
        <a href="/logout">ออกจากระบบ</a>
        {% endif %}
        {% if not is_logged_in %}
        <a href="/login">เข้าสู่ระบบ</a>
        {% endif %}
      </div>
    </div>
  </nav>
  
  <main class="container">
    <h1>ยินดีต้อนรับ!</h1>
    <p>นี่คือ Web Application ที่สร้างด้วย Lazarus Pascal</p>
  </main>
  
  <footer>
    <p>&copy; {{ year }} MyApp</p>
  </footer>
</body>
</html>
```

---

## แบบฝึกหัด

### ข้อ 1 - Blog Platform
สร้าง Blog Platform:
- หน้าแรก: รายการโพสต์ทั้งหมด (Pagination)
- หน้าโพสต์: แสดง Content พร้อม Comment
- Admin: จัดการโพสต์ (CRUD)
- Authentication: Login/Register

### ข้อ 2 - File Manager
สร้าง Web-based File Manager:
- Browse ไฟล์และโฟลเดอร์
- อัปโหลดไฟล์
- ดาวน์โหลดไฟล์
- ลบ/เปลี่ยนชื่อ

### ข้อ 3 - URL Shortener
สร้าง URL Shortener:
- รับ URL ยาว สร้าง Short URL
- Redirect ไปยัง URL จริง
- นับจำนวนคลิก
- แสดง Statistics

### ข้อ 4 - Simple CMS
สร้าง Content Management System:
- จัดการหน้าเว็บ
- Editor WYSIWYG (หรือ Markdown)
- Media Library
- User Roles

### ข้อ 5 - Chat Application
สร้าง Web Chat (Long Polling หรือ SSE):
- ห้องสนทนา
- ส่ง/รับข้อความ Real-time
- แสดง Online Users
