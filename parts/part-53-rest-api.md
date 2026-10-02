# Part 53 - REST API Development ด้วย Lazarus/Pascal

## บทนำ

REST (Representational State Transfer) เป็นสถาปัตยกรรมสำหรับการสร้าง Web Service ที่ใช้งานกันอย่างแพร่หลาย ใน Lazarus เราสามารถสร้าง REST API ได้ด้วย:
- **fpWeb** - มาพร้อมกับ Lazarus
- **Brook Framework** - Framework ที่มีประสิทธิภาพสูง
- **mORMot** - Framework ครบวงจร

---

## หลักการ REST

### HTTP Methods
| Method | ใช้สำหรับ | ตัวอย่าง |
|--------|-----------|----------|
| GET | อ่านข้อมูล | GET /users |
| POST | สร้างข้อมูลใหม่ | POST /users |
| PUT | อัปเดตข้อมูลทั้งหมด | PUT /users/1 |
| PATCH | อัปเดตบางส่วน | PATCH /users/1 |
| DELETE | ลบข้อมูล | DELETE /users/1 |

### HTTP Status Codes
| Code | ความหมาย |
|------|----------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 422 | Unprocessable Entity |
| 500 | Internal Server Error |

---

## Todo API - ตัวอย่างสมบูรณ์

### โครงสร้างโปรเจกต์

```
TodoAPI/
├── src/
│   ├── models/
│   │   └── TodoModel.pas
│   ├── handlers/
│   │   └── TodoHandler.pas
│   ├── middleware/
│   │   ├── AuthMiddleware.pas
│   │   └── LogMiddleware.pas
│   ├── utils/
│   │   ├── JsonUtils.pas
│   │   └── JWTUtils.pas
│   └── main.pas
└── TodoAPI.lpr
```

### Todo Model

```pascal
unit TodoModel;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, jsonparser, Generics.Collections, DateUtils;

type
  TTodoPriority = (tpLow, tpMedium, tpHigh, tpCritical);
  TTodoStatus = (tsActive, tsCompleted, tsArchived);

  TTodo = class
  private
    FID: Integer;
    FTitle: string;
    FDescription: string;
    FCompleted: Boolean;
    FPriority: TTodoPriority;
    FStatus: TTodoStatus;
    FDueDate: TDateTime;
    FCreatedAt: TDateTime;
    FUpdatedAt: TDateTime;
    FUserID: Integer;
    FTags: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    function ToJSON: TJSONObject;
    procedure FromJSON(AJson: TJSONObject);
    procedure AddTag(const ATag: string);
    procedure RemoveTag(const ATag: string);
    function HasTag(const ATag: string): Boolean;
    
    property ID: Integer read FID write FID;
    property Title: string read FTitle write FTitle;
    property Description: string read FDescription write FDescription;
    property Completed: Boolean read FCompleted write FCompleted;
    property Priority: TTodoPriority read FPriority write FPriority;
    property Status: TTodoStatus read FStatus write FStatus;
    property DueDate: TDateTime read FDueDate write FDueDate;
    property CreatedAt: TDateTime read FCreatedAt;
    property UpdatedAt: TDateTime read FUpdatedAt write FUpdatedAt;
    property UserID: Integer read FUserID write FUserID;
    property Tags: TStringList read FTags;
  end;

  TTodoList = specialize TObjectList<TTodo>;

  TTodoFilter = record
    Status: string;    { 'all', 'active', 'completed' }
    Priority: string;  { 'low', 'medium', 'high', 'critical', 'all' }
    UserID: Integer;
    Tag: string;
    SearchText: string;
    Page: Integer;
    PageSize: Integer;
  end;

  TTodoRepository = class
  private
    FTodos: TTodoList;
    FNextID: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    function GetAll(const AFilter: TTodoFilter): TTodoList;
    function GetByID(AID: Integer): TTodo;
    function GetByUser(AUserID: Integer): TTodoList;
    function Create_(ATodo: TTodo): TTodo;
    function Update(ATodo: TTodo): Boolean;
    function Delete(AID: Integer): Boolean;
    function Count(const AFilter: TTodoFilter): Integer;
    procedure LoadSampleData;
    
    function PriorityToString(APriority: TTodoPriority): string;
    function StringToPriority(const AStr: string): TTodoPriority;
    function StatusToString(AStatus: TTodoStatus): string;
    function StringToStatus(const AStr: string): TTodoStatus;
  end;

const
  PRIORITY_NAMES: array[TTodoPriority] of string = ('low', 'medium', 'high', 'critical');
  STATUS_NAMES: array[TTodoStatus] of string = ('active', 'completed', 'archived');

implementation

{ TTodo }
constructor TTodo.Create;
begin
  FCreatedAt := Now;
  FUpdatedAt := Now;
  FStatus := tsActive;
  FPriority := tpMedium;
  FTags := TStringList.Create;
  FTags.Delimiter := ',';
end;

destructor TTodo.Destroy;
begin
  FTags.Free;
  inherited;
end;

function TTodo.ToJSON: TJSONObject;
var
  TagsArray: TJSONArray;
  I: Integer;
begin
  Result := TJSONObject.Create;
  Result.Add('id', FID);
  Result.Add('title', FTitle);
  Result.Add('description', FDescription);
  Result.Add('completed', FCompleted);
  Result.Add('priority', PRIORITY_NAMES[FPriority]);
  Result.Add('status', STATUS_NAMES[FStatus]);
  Result.Add('user_id', FUserID);
  Result.Add('created_at', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', FCreatedAt));
  Result.Add('updated_at', FormatDateTime('yyyy-mm-dd"T"hh:nn:ss', FUpdatedAt));
  
  if FDueDate > 0 then
    Result.Add('due_date', FormatDateTime('yyyy-mm-dd', FDueDate))
  else
    Result.Add('due_date', TJSONNull.Create);
  
  TagsArray := TJSONArray.Create;
  for I := 0 to FTags.Count - 1 do
    TagsArray.Add(FTags[I]);
  Result.Add('tags', TagsArray);
end;

procedure TTodo.FromJSON(AJson: TJSONObject);
var
  TagsArray: TJSONArray;
  I: Integer;
begin
  if AJson.Find('title') <> nil then
    FTitle := AJson.Get('title', '');
  if AJson.Find('description') <> nil then
    FDescription := AJson.Get('description', '');
  if AJson.Find('completed') <> nil then
    FCompleted := AJson.Get('completed', False);
  if AJson.Find('priority') <> nil then
    FPriority := StringToPriority_Helper(AJson.Get('priority', 'medium'));
  if AJson.Find('due_date') <> nil then
  begin
    try
      FDueDate := StrToDate(AJson.Get('due_date', ''), 'yyyy-mm-dd', '-');
    except
      FDueDate := 0;
    end;
  end;
  if AJson.Find('tags') <> nil then
  begin
    FTags.Clear;
    TagsArray := AJson.Arrays['tags'];
    for I := 0 to TagsArray.Count - 1 do
      FTags.Add(TagsArray.Strings[I]);
  end;
  FUpdatedAt := Now;
end;

function TTodo.StringToPriority_Helper(const S: string): TTodoPriority;
var
  P: TTodoPriority;
begin
  for P := Low(TTodoPriority) to High(TTodoPriority) do
    if PRIORITY_NAMES[P] = LowerCase(S) then
    begin
      Result := P;
      Exit;
    end;
  Result := tpMedium;
end;

procedure TTodo.AddTag(const ATag: string);
begin
  if not HasTag(ATag) then
    FTags.Add(Trim(ATag));
end;

procedure TTodo.RemoveTag(const ATag: string);
var
  Idx: Integer;
begin
  Idx := FTags.IndexOf(ATag);
  if Idx >= 0 then FTags.Delete(Idx);
end;

function TTodo.HasTag(const ATag: string): Boolean;
begin
  Result := FTags.IndexOf(ATag) >= 0;
end;

{ TTodoRepository }
constructor TTodoRepository.Create;
begin
  FTodos := TTodoList.Create(True);
  FNextID := 1;
end;

destructor TTodoRepository.Destroy;
begin
  FTodos.Free;
  inherited;
end;

function TTodoRepository.PriorityToString(APriority: TTodoPriority): string;
begin
  Result := PRIORITY_NAMES[APriority];
end;

function TTodoRepository.StringToPriority(const AStr: string): TTodoPriority;
var
  P: TTodoPriority;
begin
  for P := Low(TTodoPriority) to High(TTodoPriority) do
    if PRIORITY_NAMES[P] = LowerCase(AStr) then
    begin
      Result := P;
      Exit;
    end;
  Result := tpMedium;
end;

function TTodoRepository.StatusToString(AStatus: TTodoStatus): string;
begin
  Result := STATUS_NAMES[AStatus];
end;

function TTodoRepository.StringToStatus(const AStr: string): TTodoStatus;
var
  S: TTodoStatus;
begin
  for S := Low(TTodoStatus) to High(TTodoStatus) do
    if STATUS_NAMES[S] = LowerCase(AStr) then
    begin
      Result := S;
      Exit;
    end;
  Result := tsActive;
end;

function TTodoRepository.GetAll(const AFilter: TTodoFilter): TTodoList;
var
  T: TTodo;
  FilteredList: TTodoList;
  StartIdx, EndIdx, I: Integer;
begin
  FilteredList := TTodoList.Create(False);
  
  for T in FTodos do
  begin
    { Filter by UserID }
    if (AFilter.UserID > 0) and (T.UserID <> AFilter.UserID) then
      Continue;
    
    { Filter by Status }
    if (AFilter.Status <> '') and (AFilter.Status <> 'all') then
      if StatusToString(T.Status) <> AFilter.Status then Continue;
    
    { Filter by Priority }
    if (AFilter.Priority <> '') and (AFilter.Priority <> 'all') then
      if PriorityToString(T.Priority) <> AFilter.Priority then Continue;
    
    { Filter by Tag }
    if AFilter.Tag <> '' then
      if not T.HasTag(AFilter.Tag) then Continue;
    
    { Filter by Search Text }
    if AFilter.SearchText <> '' then
      if (Pos(LowerCase(AFilter.SearchText), LowerCase(T.Title)) = 0) and
         (Pos(LowerCase(AFilter.SearchText), LowerCase(T.Description)) = 0) then
        Continue;
    
    FilteredList.Add(T);
  end;
  
  { Pagination }
  if (AFilter.Page > 0) and (AFilter.PageSize > 0) then
  begin
    Result := TTodoList.Create(False);
    StartIdx := (AFilter.Page - 1) * AFilter.PageSize;
    EndIdx := Min(StartIdx + AFilter.PageSize - 1, FilteredList.Count - 1);
    for I := StartIdx to EndIdx do
      if I < FilteredList.Count then
        Result.Add(FilteredList[I]);
    FilteredList.Free;
  end
  else
  begin
    Result := FilteredList;
  end;
end;

function TTodoRepository.GetByID(AID: Integer): TTodo;
var
  T: TTodo;
begin
  Result := nil;
  for T in FTodos do
    if T.ID = AID then
    begin
      Result := T;
      Exit;
    end;
end;

function TTodoRepository.GetByUser(AUserID: Integer): TTodoList;
var
  Filter: TTodoFilter;
begin
  FillChar(Filter, SizeOf(Filter), 0);
  Filter.UserID := AUserID;
  Result := GetAll(Filter);
end;

function TTodoRepository.Create_(ATodo: TTodo): TTodo;
begin
  ATodo.ID := FNextID;
  Inc(FNextID);
  FTodos.Add(ATodo);
  Result := ATodo;
end;

function TTodoRepository.Update(ATodo: TTodo): Boolean;
var
  Existing: TTodo;
begin
  Existing := GetByID(ATodo.ID);
  if Existing <> nil then
  begin
    Existing.Title := ATodo.Title;
    Existing.Description := ATodo.Description;
    Existing.Completed := ATodo.Completed;
    Existing.Priority := ATodo.Priority;
    Existing.Status := ATodo.Status;
    Existing.DueDate := ATodo.DueDate;
    Existing.UpdatedAt := Now;
    Result := True;
  end
  else
    Result := False;
end;

function TTodoRepository.Delete(AID: Integer): Boolean;
var
  I: Integer;
begin
  Result := False;
  for I := 0 to FTodos.Count - 1 do
    if FTodos[I].ID = AID then
    begin
      FTodos.Delete(I);
      Result := True;
      Exit;
    end;
end;

function TTodoRepository.Count(const AFilter: TTodoFilter): Integer;
var
  Todos: TTodoList;
begin
  Todos := GetAll(AFilter);
  Result := Todos.Count;
  Todos.Free;
end;

procedure TTodoRepository.LoadSampleData;
var
  T: TTodo;
begin
  T := TTodo.Create;
  T.Title := 'ทำรายงานสรุปประจำสัปดาห์';
  T.Description := 'จัดทำรายงานผลการดำเนินงานประจำสัปดาห์';
  T.Priority := tpHigh;
  T.UserID := 1;
  T.AddTag('งาน');
  T.AddTag('รายงาน');
  Create_(T);

  T := TTodo.Create;
  T.Title := 'ซื้อของชำ';
  T.Description := 'ข้าว, ไข่, นม, ผัก';
  T.Priority := tpMedium;
  T.UserID := 1;
  T.AddTag('ส่วนตัว');
  Create_(T);

  T := TTodo.Create;
  T.Title := 'ออกกำลังกาย';
  T.Description := 'วิ่ง 5 กิโลเมตร';
  T.Priority := tpLow;
  T.UserID := 1;
  T.AddTag('สุขภาพ');
  Create_(T);

  T := TTodo.Create;
  T.Title := 'นำเสนอโปรเจกต์';
  T.Description := 'เตรียม Slide สำหรับการนำเสนอโปรเจกต์';
  T.Priority := tpCritical;
  T.UserID := 2;
  T.AddTag('งาน');
  T.AddTag('การนำเสนอ');
  Create_(T);
end;

end.
```

---

### API Handler (fpWeb)

```pascal
unit TodoHandler;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpHTTP, fpWeb, fpjson, jsonparser,
  TodoModel;

type
  { Response Helper }
  TAPIResponse = class
  public
    class function Success(AData: TJSONData; AMessage: string = ''): TJSONObject;
    class function Error(const AMessage: string; ACode: Integer = 400): TJSONObject;
    class function Paginated(AItems: TJSONArray; ATotal, APage, APageSize: Integer): TJSONObject;
  end;

  { Base API Handler }
  TBaseAPIHandler = class(TFPHTTPModule)
  protected
    procedure WriteJSON(AResponse: TResponse; AJson: TJSONData; AStatusCode: Integer = 200);
    procedure WriteError(AResponse: TResponse; const AMessage: string; AStatusCode: Integer = 400);
    function ParseBody(ARequest: TRequest): TJSONObject;
    function GetBearerToken(ARequest: TRequest): string;
    function ValidateToken(const AToken: string; out AUserID: Integer): Boolean;
    function GetIntParam(ARequest: TRequest; const AName: string; ADefault: Integer = 0): Integer;
    function GetStringParam(ARequest: TRequest; const AName: string; const ADefault: string = ''): string;
    procedure SetCORSHeaders(AResponse: TResponse);
  end;

  { Todo Handler }
  TTodoHandler = class(TBaseAPIHandler)
  private
    FRepo: TTodoRepository;
    
    procedure HandleGetAll(ARequest: TRequest; AResponse: TResponse);
    procedure HandleGetOne(ARequest: TRequest; AResponse: TResponse; AID: Integer);
    procedure HandleCreate(ARequest: TRequest; AResponse: TResponse);
    procedure HandleUpdate(ARequest: TRequest; AResponse: TResponse; AID: Integer);
    procedure HandlePatch(ARequest: TRequest; AResponse: TResponse; AID: Integer);
    procedure HandleDelete(ARequest: TRequest; AResponse: TResponse; AID: Integer);
    procedure HandleBulkComplete(ARequest: TRequest; AResponse: TResponse);
    
    function ValidateTodo(AJson: TJSONObject; out AErrors: TStringList): Boolean;
    function TodoListToJSON(ATodos: TTodoList): TJSONArray;
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    procedure HandleRequest(ARequest: TRequest; AResponse: TResponse); override;
  end;

  { Auth Handler }
  TAuthHandler = class(TBaseAPIHandler)
  private
    procedure HandleLogin(ARequest: TRequest; AResponse: TResponse);
    procedure HandleRegister(ARequest: TRequest; AResponse: TResponse);
    procedure HandleRefreshToken(ARequest: TRequest; AResponse: TResponse);
    procedure HandleLogout(ARequest: TRequest; AResponse: TResponse);
  public
    procedure HandleRequest(ARequest: TRequest; AResponse: TResponse); override;
  end;

implementation

{ TAPIResponse }
class function TAPIResponse.Success(AData: TJSONData; AMessage: string = ''): TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('success', True);
  if AMessage <> '' then
    Result.Add('message', AMessage);
  if AData <> nil then
    Result.Add('data', AData);
end;

class function TAPIResponse.Error(const AMessage: string; ACode: Integer = 400): TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('success', False);
  Result.Add('error', AMessage);
  Result.Add('code', ACode);
end;

class function TAPIResponse.Paginated(AItems: TJSONArray; ATotal, APage, APageSize: Integer): TJSONObject;
var
  Meta: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.Add('success', True);
  Result.Add('data', AItems);
  
  Meta := TJSONObject.Create;
  Meta.Add('total', ATotal);
  Meta.Add('page', APage);
  Meta.Add('page_size', APageSize);
  Meta.Add('total_pages', Ceil(ATotal / APageSize));
  Meta.Add('has_next', APage * APageSize < ATotal);
  Meta.Add('has_prev', APage > 1);
  Result.Add('meta', Meta);
end;

{ TBaseAPIHandler }
procedure TBaseAPIHandler.WriteJSON(AResponse: TResponse; AJson: TJSONData; AStatusCode: Integer);
begin
  AResponse.Code := AStatusCode;
  AResponse.ContentType := 'application/json; charset=utf-8';
  SetCORSHeaders(AResponse);
  AResponse.Content := AJson.AsJSON;
  AJson.Free;
end;

procedure TBaseAPIHandler.WriteError(AResponse: TResponse; const AMessage: string; AStatusCode: Integer);
begin
  WriteJSON(AResponse, TAPIResponse.Error(AMessage, AStatusCode), AStatusCode);
end;

function TBaseAPIHandler.ParseBody(ARequest: TRequest): TJSONObject;
var
  Parser: TJSONParser;
  Data: TJSONData;
begin
  Result := nil;
  if ARequest.Content = '' then Exit;
  
  Parser := TJSONParser.Create(ARequest.Content, []);
  try
    Data := Parser.Parse;
    if Data is TJSONObject then
      Result := TJSONObject(Data)
    else
    begin
      Data.Free;
      raise Exception.Create('Request body ต้องเป็น JSON Object');
    end;
  finally
    Parser.Free;
  end;
end;

function TBaseAPIHandler.GetBearerToken(ARequest: TRequest): string;
var
  AuthHeader: string;
begin
  AuthHeader := ARequest.GetFieldByName('Authorization');
  if Copy(AuthHeader, 1, 7) = 'Bearer ' then
    Result := Copy(AuthHeader, 8, MaxInt)
  else
    Result := '';
end;

function TBaseAPIHandler.ValidateToken(const AToken: string; out AUserID: Integer): Boolean;
begin
  { TODO: Implement JWT validation }
  { ตัวอย่างง่ายๆ }
  AUserID := 1;
  Result := AToken <> '';
end;

function TBaseAPIHandler.GetIntParam(ARequest: TRequest; const AName: string; ADefault: Integer): Integer;
begin
  Result := StrToIntDef(ARequest.QueryFields.Values[AName], ADefault);
end;

function TBaseAPIHandler.GetStringParam(ARequest: TRequest; const AName: string; const ADefault: string): string;
begin
  Result := ARequest.QueryFields.Values[AName];
  if Result = '' then Result := ADefault;
end;

procedure TBaseAPIHandler.SetCORSHeaders(AResponse: TResponse);
begin
  AResponse.SetFieldByName('Access-Control-Allow-Origin', '*');
  AResponse.SetFieldByName('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
  AResponse.SetFieldByName('Access-Control-Allow-Headers', 'Content-Type, Authorization');
end;

{ TTodoHandler }
constructor TTodoHandler.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  FRepo := TTodoRepository.Create;
  FRepo.LoadSampleData;
end;

destructor TTodoHandler.Destroy;
begin
  FRepo.Free;
  inherited;
end;

procedure TTodoHandler.HandleRequest(ARequest: TRequest; AResponse: TResponse);
var
  PathParts: TStringArray;
  ID: Integer;
  Method: string;
begin
  Method := UpperCase(ARequest.Method);
  
  { Handle CORS Preflight }
  if Method = 'OPTIONS' then
  begin
    SetCORSHeaders(AResponse);
    AResponse.Code := 204;
    Exit;
  end;
  
  { Parse path: /api/todos/{id} }
  PathParts := ARequest.PathInfo.Split(['/']);
  { PathParts[0] = '', [1] = 'api', [2] = 'todos', [3] = id (optional) }
  
  ID := 0;
  if Length(PathParts) >= 4 then
    ID := StrToIntDef(PathParts[3], 0);
  
  { Route }
  if (Length(PathParts) >= 4) and (PathParts[3] = 'bulk-complete') then
  begin
    if Method = 'POST' then HandleBulkComplete(ARequest, AResponse)
    else WriteError(AResponse, 'Method tidak diizinkan', 405);
  end
  else if ID > 0 then
  begin
    case Method of
      'GET':    HandleGetOne(ARequest, AResponse, ID);
      'PUT':    HandleUpdate(ARequest, AResponse, ID);
      'PATCH':  HandlePatch(ARequest, AResponse, ID);
      'DELETE': HandleDelete(ARequest, AResponse, ID);
    else
      WriteError(AResponse, 'Method ไม่รองรับ', 405);
    end;
  end
  else
  begin
    case Method of
      'GET':  HandleGetAll(ARequest, AResponse);
      'POST': HandleCreate(ARequest, AResponse);
    else
      WriteError(AResponse, 'Method ไม่รองรับ', 405);
    end;
  end;
end;

procedure TTodoHandler.HandleGetAll(ARequest: TRequest; AResponse: TResponse);
var
  Filter: TTodoFilter;
  Todos: TTodoList;
  JsonArray: TJSONArray;
  Total: Integer;
  Token: string;
  UserID: Integer;
begin
  { Authentication }
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  FillChar(Filter, SizeOf(Filter), 0);
  Filter.UserID := UserID;
  Filter.Status := GetStringParam(ARequest, 'status', 'all');
  Filter.Priority := GetStringParam(ARequest, 'priority', 'all');
  Filter.Tag := GetStringParam(ARequest, 'tag');
  Filter.SearchText := GetStringParam(ARequest, 'q');
  Filter.Page := GetIntParam(ARequest, 'page', 1);
  Filter.PageSize := GetIntParam(ARequest, 'per_page', 10);
  
  { คำนวณ Total ก่อน Pagination }
  Total := FRepo.Count(Filter);
  
  Todos := FRepo.GetAll(Filter);
  try
    JsonArray := TodoListToJSON(Todos);
    WriteJSON(AResponse, TAPIResponse.Paginated(JsonArray, Total, Filter.Page, Filter.PageSize));
  finally
    Todos.Free;
  end;
end;

procedure TTodoHandler.HandleGetOne(ARequest: TRequest; AResponse: TResponse; AID: Integer);
var
  Todo: TTodo;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  Todo := FRepo.GetByID(AID);
  if Todo = nil then
  begin
    WriteError(AResponse, 'ไม่พบ Todo ที่ระบุ', 404);
    Exit;
  end;
  
  { ตรวจสอบว่าเป็นของ User นั้นๆ }
  if Todo.UserID <> UserID then
  begin
    WriteError(AResponse, 'Forbidden', 403);
    Exit;
  end;
  
  WriteJSON(AResponse, TAPIResponse.Success(Todo.ToJSON));
end;

procedure TTodoHandler.HandleCreate(ARequest: TRequest; AResponse: TResponse);
var
  Body: TJSONObject;
  NewTodo: TTodo;
  Errors: TStringList;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  try
    Body := ParseBody(ARequest);
  except
    on E: Exception do
    begin
      WriteError(AResponse, 'Invalid JSON: ' + E.Message);
      Exit;
    end;
  end;
  
  if Body = nil then
  begin
    WriteError(AResponse, 'ต้องระบุข้อมูลใน Request Body');
    Exit;
  end;
  
  try
    if not ValidateTodo(Body, Errors) then
    begin
      WriteJSON(AResponse, 
        TAPIResponse.Error('ข้อมูลไม่ถูกต้อง: ' + Errors.CommaText), 422);
      Errors.Free;
      Exit;
    end;
    Errors.Free;
    
    NewTodo := TTodo.Create;
    NewTodo.FromJSON(Body);
    NewTodo.UserID := UserID;
    FRepo.Create_(NewTodo);
    
    WriteJSON(AResponse, 
      TAPIResponse.Success(NewTodo.ToJSON, 'สร้าง Todo เรียบร้อย'), 201);
  finally
    Body.Free;
  end;
end;

procedure TTodoHandler.HandleUpdate(ARequest: TRequest; AResponse: TResponse; AID: Integer);
var
  Body: TJSONObject;
  Todo: TTodo;
  Errors: TStringList;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  Todo := FRepo.GetByID(AID);
  if Todo = nil then
  begin
    WriteError(AResponse, 'ไม่พบ Todo ที่ระบุ', 404);
    Exit;
  end;
  
  if Todo.UserID <> UserID then
  begin
    WriteError(AResponse, 'Forbidden', 403);
    Exit;
  end;
  
  try
    Body := ParseBody(ARequest);
    if Body = nil then
    begin
      WriteError(AResponse, 'ต้องระบุข้อมูลใน Request Body');
      Exit;
    end;
    
    if not ValidateTodo(Body, Errors) then
    begin
      WriteJSON(AResponse, TAPIResponse.Error('ข้อมูลไม่ถูกต้อง'), 422);
      Errors.Free;
      Exit;
    end;
    Errors.Free;
    
    Todo.FromJSON(Body);
    FRepo.Update(Todo);
    
    WriteJSON(AResponse, TAPIResponse.Success(Todo.ToJSON, 'อัปเดต Todo เรียบร้อย'));
  finally
    Body.Free;
  end;
end;

procedure TTodoHandler.HandlePatch(ARequest: TRequest; AResponse: TResponse; AID: Integer);
var
  Body: TJSONObject;
  Todo: TTodo;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  Todo := FRepo.GetByID(AID);
  if Todo = nil then
  begin
    WriteError(AResponse, 'ไม่พบ Todo ที่ระบุ', 404);
    Exit;
  end;
  
  if Todo.UserID <> UserID then
  begin
    WriteError(AResponse, 'Forbidden', 403);
    Exit;
  end;
  
  try
    Body := ParseBody(ARequest);
    if Body = nil then
    begin
      WriteError(AResponse, 'ต้องระบุข้อมูลใน Request Body');
      Exit;
    end;
    
    { Partial update - อัปเดตเฉพาะฟิลด์ที่ส่งมา }
    if Body.Find('title') <> nil then
      Todo.Title := Body.Get('title', '');
    if Body.Find('description') <> nil then
      Todo.Description := Body.Get('description', '');
    if Body.Find('completed') <> nil then
      Todo.Completed := Body.Get('completed', False);
    if Body.Find('priority') <> nil then
      Todo.Priority := FRepo.StringToPriority(Body.Get('priority', 'medium'));
    
    Todo.UpdatedAt := Now;
    FRepo.Update(Todo);
    
    WriteJSON(AResponse, TAPIResponse.Success(Todo.ToJSON, 'อัปเดต Todo เรียบร้อย'));
  finally
    Body.Free;
  end;
end;

procedure TTodoHandler.HandleDelete(ARequest: TRequest; AResponse: TResponse; AID: Integer);
var
  Todo: TTodo;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  Todo := FRepo.GetByID(AID);
  if Todo = nil then
  begin
    WriteError(AResponse, 'ไม่พบ Todo ที่ระบุ', 404);
    Exit;
  end;
  
  if Todo.UserID <> UserID then
  begin
    WriteError(AResponse, 'Forbidden', 403);
    Exit;
  end;
  
  FRepo.Delete(AID);
  WriteJSON(AResponse, TAPIResponse.Success(nil, 'ลบ Todo เรียบร้อย'), 200);
end;

procedure TTodoHandler.HandleBulkComplete(ARequest: TRequest; AResponse: TResponse);
var
  Body: TJSONObject;
  IDs: TJSONArray;
  I, ID, Count: Integer;
  Todo: TTodo;
  Token: string;
  UserID: Integer;
begin
  Token := GetBearerToken(ARequest);
  if not ValidateToken(Token, UserID) then
  begin
    WriteError(AResponse, 'Unauthorized', 401);
    Exit;
  end;
  
  try
    Body := ParseBody(ARequest);
    if (Body = nil) or (Body.Find('ids') = nil) then
    begin
      WriteError(AResponse, 'ต้องระบุ ids');
      Exit;
    end;
    
    IDs := Body.Arrays['ids'];
    Count := 0;
    
    for I := 0 to IDs.Count - 1 do
    begin
      ID := IDs.Integers[I];
      Todo := FRepo.GetByID(ID);
      if (Todo <> nil) and (Todo.UserID = UserID) then
      begin
        Todo.Completed := True;
        Todo.Status := tsCompleted;
        Todo.UpdatedAt := Now;
        FRepo.Update(Todo);
        Inc(Count);
      end;
    end;
    
    WriteJSON(AResponse, TAPIResponse.Success(
      nil, Format('อัปเดต %d Todo เรียบร้อย', [Count])));
  finally
    Body.Free;
  end;
end;

function TTodoHandler.ValidateTodo(AJson: TJSONObject; out AErrors: TStringList): Boolean;
begin
  AErrors := TStringList.Create;
  
  if AJson.Find('title') = nil then
    AErrors.Add('title is required')
  else if Trim(AJson.Get('title', '')) = '' then
    AErrors.Add('title must not be empty')
  else if Length(AJson.Get('title', '')) > 255 then
    AErrors.Add('title must not exceed 255 characters');
  
  Result := AErrors.Count = 0;
end;

function TTodoHandler.TodoListToJSON(ATodos: TTodoList): TJSONArray;
var
  T: TTodo;
begin
  Result := TJSONArray.Create;
  for T in ATodos do
    Result.Add(T.ToJSON);
end;

end.
```

---

### JWT Utilities

```pascal
unit JWTUtils;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, base64, fpjson, jsonparser, DateUtils,
  HMACUtils;  { Custom unit }

type
  TJWTClaims = record
    UserID: Integer;
    Username: string;
    Role: string;
    ExpiresAt: TDateTime;
    IssuedAt: TDateTime;
  end;

  TJWT = class
  private
    class var FSecretKey: string;
    class function Base64URLEncode(const AData: string): string;
    class function Base64URLDecode(const AData: string): string;
    class function CreateHeader: string;
    class function CreatePayload(const AClaims: TJWTClaims): string;
    class function CreateSignature(const AHeader, APayload: string): string;
  public
    class procedure SetSecretKey(const AKey: string);
    class function CreateToken(const AClaims: TJWTClaims): string;
    class function ValidateToken(const AToken: string; out AClaims: TJWTClaims): Boolean;
    class function ExtractUserID(const AToken: string): Integer;
    class function IsExpired(const AClaims: TJWTClaims): Boolean;
  end;

implementation

class function TJWT.Base64URLEncode(const AData: string): string;
begin
  Result := EncodeStringBase64(AData);
  { แปลง Base64 เป็น Base64URL }
  Result := StringReplace(Result, '+', '-', [rfReplaceAll]);
  Result := StringReplace(Result, '/', '_', [rfReplaceAll]);
  Result := StringReplace(Result, '=', '', [rfReplaceAll]);
end;

class function TJWT.Base64URLDecode(const AData: string): string;
var
  Padded: string;
begin
  Padded := AData;
  Padded := StringReplace(Padded, '-', '+', [rfReplaceAll]);
  Padded := StringReplace(Padded, '_', '/', [rfReplaceAll]);
  { เพิ่ม Padding }
  case Length(Padded) mod 4 of
    2: Padded := Padded + '==';
    3: Padded := Padded + '=';
  end;
  Result := DecodeStringBase64(Padded);
end;

class function TJWT.CreateHeader: string;
var
  Header: TJSONObject;
begin
  Header := TJSONObject.Create;
  try
    Header.Add('alg', 'HS256');
    Header.Add('typ', 'JWT');
    Result := Base64URLEncode(Header.AsJSON);
  finally
    Header.Free;
  end;
end;

class function TJWT.CreatePayload(const AClaims: TJWTClaims): string;
var
  Payload: TJSONObject;
begin
  Payload := TJSONObject.Create;
  try
    Payload.Add('sub', AClaims.UserID);
    Payload.Add('username', AClaims.Username);
    Payload.Add('role', AClaims.Role);
    Payload.Add('exp', DateTimeToUnix(AClaims.ExpiresAt));
    Payload.Add('iat', DateTimeToUnix(AClaims.IssuedAt));
    Result := Base64URLEncode(Payload.AsJSON);
  finally
    Payload.Free;
  end;
end;

class function TJWT.CreateSignature(const AHeader, APayload: string): string;
var
  Data: string;
  Hash: string;
begin
  Data := AHeader + '.' + APayload;
  { ใช้ HMAC-SHA256 }
  Hash := HMACSHA256(Data, FSecretKey);
  Result := Base64URLEncode(Hash);
end;

class procedure TJWT.SetSecretKey(const AKey: string);
begin
  FSecretKey := AKey;
end;

class function TJWT.CreateToken(const AClaims: TJWTClaims): string;
var
  Header, Payload, Signature: string;
begin
  Header := CreateHeader;
  Payload := CreatePayload(AClaims);
  Signature := CreateSignature(Header, Payload);
  Result := Header + '.' + Payload + '.' + Signature;
end;

class function TJWT.ValidateToken(const AToken: string; out AClaims: TJWTClaims): Boolean;
var
  Parts: TStringArray;
  ExpectedSig, ActualSig: string;
  PayloadJson: string;
  Payload: TJSONObject;
  Parser: TJSONParser;
begin
  Result := False;
  FillChar(AClaims, SizeOf(AClaims), 0);
  
  Parts := AToken.Split(['.']);
  if Length(Parts) <> 3 then Exit;
  
  { ตรวจสอบ Signature }
  ExpectedSig := CreateSignature(Parts[0], Parts[1]);
  if ExpectedSig <> Parts[2] then Exit;
  
  { Decode Payload }
  PayloadJson := Base64URLDecode(Parts[1]);
  Parser := TJSONParser.Create(PayloadJson, []);
  try
    Payload := TJSONObject(Parser.Parse);
    try
      AClaims.UserID := Payload.Get('sub', 0);
      AClaims.Username := Payload.Get('username', '');
      AClaims.Role := Payload.Get('role', '');
      AClaims.ExpiresAt := UnixToDateTime(Payload.Get('exp', Int64(0)));
      AClaims.IssuedAt := UnixToDateTime(Payload.Get('iat', Int64(0)));
    finally
      Payload.Free;
    end;
  finally
    Parser.Free;
  end;
  
  { ตรวจสอบหมดอายุ }
  if IsExpired(AClaims) then Exit;
  
  Result := True;
end;

class function TJWT.ExtractUserID(const AToken: string): Integer;
var
  Claims: TJWTClaims;
begin
  if ValidateToken(AToken, Claims) then
    Result := Claims.UserID
  else
    Result := 0;
end;

class function TJWT.IsExpired(const AClaims: TJWTClaims): Boolean;
begin
  Result := Now > AClaims.ExpiresAt;
end;

end.
```

---

### Main Program

```pascal
program TodoAPI;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, fpHTTPApp, fpWeb, 
  TodoHandler;

var
  App: TFPHTTPApplication;
  Handler: TTodoHandler;

procedure ConfigureRoutes;
begin
  Handler := TTodoHandler.Create(Application);
  Handler.ModuleName := 'todos';
  
  { fpWeb จะ Route /api/todos/* ไปยัง Handler }
end;

begin
  App := TFPHTTPApplication.Create(nil);
  try
    App.Threaded := True;
    App.Port := 8080;
    App.OnLog := nil;
    
    ConfigureRoutes;
    
    WriteLn('Todo API Server กำลังทำงานที่ port 8080...');
    WriteLn('กด Ctrl+C เพื่อหยุดเซิร์ฟเวอร์');
    
    App.Run;
  finally
    App.Free;
  end;
end.
```

---

## การทดสอบด้วย curl

```bash
# สร้าง Todo
curl -X POST http://localhost:8080/api/todos \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_token_here" \
  -d '{
    "title": "เรียน Pascal",
    "description": "อ่านหนังสือ Lazarus Programming",
    "priority": "high"
  }'

# ดูรายการทั้งหมด
curl http://localhost:8080/api/todos \
  -H "Authorization: Bearer your_token_here"

# ดู Todo เดียว
curl http://localhost:8080/api/todos/1 \
  -H "Authorization: Bearer your_token_here"

# อัปเดต
curl -X PUT http://localhost:8080/api/todos/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_token_here" \
  -d '{"title": "อัปเดตชื่อ", "completed": true}'

# ลบ
curl -X DELETE http://localhost:8080/api/todos/1 \
  -H "Authorization: Bearer your_token_here"

# ค้นหา
curl "http://localhost:8080/api/todos?q=Pascal&status=active&page=1&per_page=5" \
  -H "Authorization: Bearer your_token_here"
```

---

## แบบฝึกหัด

### ข้อ 1 - User API
สร้าง User API ที่รองรับ:
- POST /api/auth/register
- POST /api/auth/login
- GET /api/users/me
- PUT /api/users/me
- DELETE /api/users/me

### ข้อ 2 - Rate Limiting
เพิ่ม Rate Limiting Middleware ที่จำกัด Request เป็น 100 ครั้ง/นาที/IP

### ข้อ 3 - Blog API
สร้าง Blog API:
- Posts CRUD
- Comments (nested)
- Tags
- Categories
- Pagination

### ข้อ 4 - File Upload API
สร้าง API สำหรับอัปโหลดไฟล์:
- POST /api/files (multipart/form-data)
- GET /api/files/{id}
- DELETE /api/files/{id}
- ตรวจสอบ File Type และขนาด

### ข้อ 5 - WebHook System
สร้าง WebHook System:
- Register webhook endpoints
- Trigger events
- Retry mechanism
- Signature verification
