# Part 40 - HTTP Client

## บทนำ

HTTP (HyperText Transfer Protocol) เป็น Protocol หลักของ World Wide Web ในการพัฒนาโปรแกรมสมัยใหม่ เราต้องการส่ง HTTP Requests ไปยัง API, Web Services หรือ Websites ต่างๆ Lazarus มีหลายวิธีในการทำ HTTP requests: Indy (TIdHTTP), Synapse (HttpSend), และ FCL-Web

---

## 40.1 ภาพรวม HTTP Protocol

### HTTP Methods

```
GET    - ดึงข้อมูล (Read)
POST   - ส่งข้อมูลใหม่ (Create)
PUT    - แทนที่ข้อมูล (Update/Replace)
PATCH  - แก้ไขข้อมูลบางส่วน (Update/Partial)
DELETE - ลบข้อมูล (Delete)
HEAD   - เหมือน GET แต่ไม่ส่ง Body กลับ
OPTIONS- ดู Methods ที่รองรับ
```

### HTTP Status Codes

```
2xx - Success
  200 OK              - สำเร็จ
  201 Created         - สร้างข้อมูลสำเร็จ
  204 No Content      - สำเร็จแต่ไม่มีข้อมูลส่งกลับ

3xx - Redirection
  301 Moved Permanently   - ย้ายถาวร
  302 Found              - ย้ายชั่วคราว
  304 Not Modified       - ข้อมูลไม่เปลี่ยน (ใช้ Cache)

4xx - Client Error
  400 Bad Request        - Request ไม่ถูกต้อง
  401 Unauthorized       - ต้อง Authenticate
  403 Forbidden          - ไม่มีสิทธิ์
  404 Not Found          - ไม่พบ Resource
  405 Method Not Allowed - Method ไม่รองรับ
  429 Too Many Requests  - Request มากเกินไป

5xx - Server Error
  500 Internal Server Error - Server Error
  502 Bad Gateway           - Gateway Error
  503 Service Unavailable   - Service ไม่พร้อม
  504 Gateway Timeout       - Timeout
```

---

## 40.2 TIdHTTP (Indy)

TIdHTTP เป็น Component ใน Indy library สำหรับ HTTP requests

```pascal
unit IdHTTPDemo;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  IdHTTP, IdSSLOpenSSL, IdComponent, IdBaseComponent;

type
  TForm1 = class(TForm)
    EditURL: TEdit;
    btnGet: TButton;
    btnPost: TButton;
    MemoResponse: TMemo;
    lblStatus: TLabel;
    ProgressBar1: TProgressBar;
    IdHTTP1: TIdHTTP;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnGetClick(Sender: TObject);
    procedure btnPostClick(Sender: TObject);
    procedure IdHTTP1WorkBegin(ASender: TObject; AWorkMode: TWorkMode; AWorkCountMax: Int64);
    procedure IdHTTP1Work(ASender: TObject; AWorkMode: TWorkMode; AWorkCount: Int64);
    procedure IdHTTP1WorkEnd(ASender: TObject; AWorkMode: TWorkMode);
  private
    FSSLHandler: TIdSSLIOHandlerSocketOpenSSL;
    procedure SetupSSL;
    procedure SetupHeaders;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  SetupSSL;
  SetupHeaders;
  
  EditURL.Text := 'https://httpbin.org/get';
  
  // ตั้งค่า HTTP
  IdHTTP1.ConnectTimeout := 10000;
  IdHTTP1.ReadTimeout := 30000;
  IdHTTP1.HandleRedirects := True;
  IdHTTP1.MaxRedirects := 5;
  IdHTTP1.AllowCookies := True;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FSSLHandler.Free;
end;

procedure TForm1.SetupSSL;
begin
  FSSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  FSSLHandler.SSLOptions.Method := sslvSSLv23;
  FSSLHandler.SSLOptions.Mode := sslmClient;
  FSSLHandler.SSLOptions.VerifyMode := [];
  FSSLHandler.SSLOptions.VerifyDepth := 9;
  IdHTTP1.IOHandler := FSSLHandler;
end;

procedure TForm1.SetupHeaders;
begin
  IdHTTP1.Request.UserAgent := 'Mozilla/5.0 (Lazarus HTTP Client 1.0)';
  IdHTTP1.Request.Accept := 'application/json, text/html, */*';
  IdHTTP1.Request.AcceptLanguage := 'th-TH, en-US, en;q=0.9';
  IdHTTP1.Request.AcceptCharSet := 'UTF-8, ISO-8859-1;q=0.9';
end;

procedure TForm1.btnGetClick(Sender: TObject);
var
  URL: string;
  Response: string;
begin
  URL := Trim(EditURL.Text);
  if URL = '' then Exit;
  
  MemoResponse.Clear;
  lblStatus.Caption := 'กำลังส่ง Request...';
  Application.ProcessMessages;
  
  try
    Response := IdHTTP1.Get(URL);
    
    lblStatus.Caption := Format('สำเร็จ: HTTP %d', [IdHTTP1.ResponseCode]);
    MemoResponse.Lines.Add('=== Response Headers ===');
    MemoResponse.Lines.Add('Status: ' + IdHTTP1.ResponseText);
    MemoResponse.Lines.Add('Content-Type: ' + IdHTTP1.Response.ContentType);
    MemoResponse.Lines.Add('Content-Length: ' + IntToStr(Length(Response)));
    MemoResponse.Lines.Add('');
    MemoResponse.Lines.Add('=== Response Body ===');
    MemoResponse.Lines.Add(Response);
    
  except
    on E: EIdHTTPProtocolException do
    begin
      lblStatus.Caption := Format('HTTP Error: %d', [E.ErrorCode]);
      MemoResponse.Lines.Add('HTTP Error: ' + E.Message);
      MemoResponse.Lines.Add('Error Content: ' + E.ErrorMessage);
    end;
    on E: Exception do
    begin
      lblStatus.Caption := 'Error: ' + E.Message;
      MemoResponse.Lines.Add('Error: ' + E.Message);
    end;
  end;
end;

procedure TForm1.btnPostClick(Sender: TObject);
var
  URL: string;
  PostData: TStringStream;
  Response: string;
begin
  URL := 'https://httpbin.org/post';
  
  PostData := TStringStream.Create('{"name":"John","age":30}', TEncoding.UTF8);
  try
    IdHTTP1.Request.ContentType := 'application/json';
    IdHTTP1.Request.CharSet := 'UTF-8';
    
    Response := IdHTTP1.Post(URL, PostData);
    
    MemoResponse.Clear;
    MemoResponse.Lines.Add('POST Response:');
    MemoResponse.Lines.Add(Response);
    
  finally
    PostData.Free;
  end;
end;

// Progress Tracking
procedure TForm1.IdHTTP1WorkBegin(ASender: TObject; AWorkMode: TWorkMode; AWorkCountMax: Int64);
begin
  ProgressBar1.Max := AWorkCountMax;
  ProgressBar1.Position := 0;
end;

procedure TForm1.IdHTTP1Work(ASender: TObject; AWorkMode: TWorkMode; AWorkCount: Int64);
begin
  ProgressBar1.Position := AWorkCount;
  Application.ProcessMessages;
end;

procedure TForm1.IdHTTP1WorkEnd(ASender: TObject; AWorkMode: TWorkMode);
begin
  ProgressBar1.Position := ProgressBar1.Max;
end;

end.
```

---

## 40.3 Synapse HTTP Client

Synapse เป็น Library ที่เบาและใช้งานง่าย

```pascal
unit SynapseHTTP;

interface

uses
  Classes, SysUtils,
  HttpSend, SynaUtil, MimeTypes;

type
  THTTPResponse = record
    StatusCode: Integer;
    StatusText: string;
    Headers: TStringList;
    Body: string;
    Error: string;
  end;
  
  TSynapseClient = class
  private
    function ExtractHeaderValue(const Headers: TStringList; 
                                 const Name: string): string;
  public
    function Get(const URL: string): THTTPResponse;
    function Post(const URL, Body, ContentType: string): THTTPResponse;
    function Put(const URL, Body: string): THTTPResponse;
    function Delete(const URL: string): THTTPResponse;
    function PostForm(const URL: string; const FormData: TStringList): THTTPResponse;
    function Download(const URL, SavePath: string): Boolean;
  end;

implementation

function TSynapseClient.ExtractHeaderValue(const Headers: TStringList; 
  const Name: string): string;
var
  i: Integer;
  Header: string;
begin
  Result := '';
  for i := 0 to Headers.Count - 1 do
  begin
    Header := Headers[i];
    if AnsiSameText(Copy(Header, 1, Length(Name) + 1), Name + ':') then
    begin
      Result := Trim(Copy(Header, Length(Name) + 2, MaxInt));
      Break;
    end;
  end;
end;

function TSynapseClient.Get(const URL: string): THTTPResponse;
var
  HTTP: THTTPSend;
begin
  Result.Headers := TStringList.Create;
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('User-Agent: Lazarus Synapse Client 1.0');
    HTTP.Headers.Add('Accept: application/json, text/html, */*');
    
    if HTTP.HTTPMethod('GET', URL) then
    begin
      Result.StatusCode := HTTP.ResultCode;
      Result.StatusText := HTTP.ResultString;
      Result.Headers.Assign(HTTP.Headers);
      
      SetString(Result.Body, PChar(HTTP.Document.Memory), HTTP.Document.Size);
      Result.Error := '';
    end else
    begin
      Result.StatusCode := 0;
      Result.Error := 'Connection failed';
    end;
  finally
    HTTP.Free;
  end;
end;

function TSynapseClient.Post(const URL, Body, ContentType: string): THTTPResponse;
var
  HTTP: THTTPSend;
begin
  Result.Headers := TStringList.Create;
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('User-Agent: Lazarus Synapse Client');
    HTTP.Headers.Add('Content-Type: ' + ContentType);
    
    // ใส่ Body
    HTTP.Document.Write(PChar(Body)^, Length(Body));
    
    if HTTP.HTTPMethod('POST', URL) then
    begin
      Result.StatusCode := HTTP.ResultCode;
      Result.StatusText := HTTP.ResultString;
      Result.Headers.Assign(HTTP.Headers);
      SetString(Result.Body, PChar(HTTP.Document.Memory), HTTP.Document.Size);
    end else
      Result.Error := 'Request failed';
  finally
    HTTP.Free;
  end;
end;

function TSynapseClient.PostForm(const URL: string; 
  const FormData: TStringList): THTTPResponse;
var
  Encoded: string;
  i: Integer;
begin
  // Encode Form Data
  Encoded := '';
  for i := 0 to FormData.Count - 1 do
  begin
    if i > 0 then Encoded := Encoded + '&';
    Encoded := Encoded + 
      EncodeURL(FormData.Names[i]) + '=' + 
      EncodeURL(FormData.ValueFromIndex[i]);
  end;
  
  Result := Post(URL, Encoded, 'application/x-www-form-urlencoded');
end;

function TSynapseClient.Put(const URL, Body: string): THTTPResponse;
var
  HTTP: THTTPSend;
begin
  Result.Headers := TStringList.Create;
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('Content-Type: application/json');
    HTTP.Document.Write(PChar(Body)^, Length(Body));
    
    if HTTP.HTTPMethod('PUT', URL) then
    begin
      Result.StatusCode := HTTP.ResultCode;
      SetString(Result.Body, PChar(HTTP.Document.Memory), HTTP.Document.Size);
    end else
      Result.Error := 'Request failed';
  finally
    HTTP.Free;
  end;
end;

function TSynapseClient.Delete(const URL: string): THTTPResponse;
var
  HTTP: THTTPSend;
begin
  Result.Headers := TStringList.Create;
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('DELETE', URL) then
    begin
      Result.StatusCode := HTTP.ResultCode;
      SetString(Result.Body, PChar(HTTP.Document.Memory), HTTP.Document.Size);
    end else
      Result.Error := 'Request failed';
  finally
    HTTP.Free;
  end;
end;

function TSynapseClient.Download(const URL, SavePath: string): Boolean;
var
  HTTP: THTTPSend;
  FS: TFileStream;
begin
  Result := False;
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('GET', URL) and (HTTP.ResultCode = 200) then
    begin
      FS := TFileStream.Create(SavePath, fmCreate);
      try
        HTTP.Document.Seek(0, soBeginning);
        FS.CopyFrom(HTTP.Document, HTTP.Document.Size);
        Result := True;
      finally
        FS.Free;
      end;
    end;
  finally
    HTTP.Free;
  end;
end;

end.
```

---

## 40.4 GET, POST, PUT, DELETE Requests

```pascal
unit RESTClient;

interface

uses
  Classes, SysUtils,
  IdHTTP, IdSSLOpenSSL,
  fpjson, jsonparser;

type
  TRESTClient = class
  private
    FHTTP: TIdHTTP;
    FSSLHandler: TIdSSLIOHandlerSocketOpenSSL;
    FBaseURL: string;
    FAuthToken: string;
    
    procedure SetupHeaders;
    function BuildURL(const Path: string): string;
    function ResponseToJSON(const Response: string): TJSONData;
  public
    constructor Create(const BaseURL: string);
    destructor Destroy; override;
    
    // Basic Auth
    procedure SetBasicAuth(const Username, Password: string);
    
    // Bearer Token Auth
    procedure SetBearerToken(const Token: string);
    
    // HTTP Methods
    function Get(const Path: string): TJSONData;
    function GetString(const Path: string): string;
    function Post(const Path: string; const Body: TJSONObject): TJSONData;
    function PostString(const Path, Body, ContentType: string): string;
    function Put(const Path: string; const Body: TJSONObject): TJSONData;
    function Patch(const Path: string; const Body: TJSONObject): TJSONData;
    function Delete(const Path: string): Boolean;
    
    // File Upload
    function UploadFile(const Path, FieldName, FileName: string): TJSONData;
    
    // Properties
    property BaseURL: string read FBaseURL write FBaseURL;
    property LastStatusCode: Integer read (FHTTP.ResponseCode);
    property LastStatusText: string read (FHTTP.ResponseText);
  end;

implementation

uses
  IdMultipartFormData;

constructor TRESTClient.Create(const BaseURL: string);
begin
  inherited Create;
  FBaseURL := BaseURL;
  FAuthToken := '';
  
  FHTTP := TIdHTTP.Create(nil);
  FHTTP.ConnectTimeout := 10000;
  FHTTP.ReadTimeout := 30000;
  FHTTP.HandleRedirects := True;
  
  FSSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  FSSLHandler.SSLOptions.Method := sslvSSLv23;
  FHTTP.IOHandler := FSSLHandler;
  
  SetupHeaders;
end;

destructor TRESTClient.Destroy;
begin
  FHTTP.Free;
  FSSLHandler.Free;
  inherited;
end;

procedure TRESTClient.SetupHeaders;
begin
  FHTTP.Request.UserAgent := 'Lazarus REST Client 1.0';
  FHTTP.Request.Accept := 'application/json';
  FHTTP.Request.AcceptCharSet := 'UTF-8';
  FHTTP.Request.ContentType := 'application/json';
end;

function TRESTClient.BuildURL(const Path: string): string;
begin
  if Path.StartsWith('http') then
    Result := Path
  else
  begin
    Result := FBaseURL;
    if not Result.EndsWith('/') and not Path.StartsWith('/') then
      Result := Result + '/';
    Result := Result + Path;
  end;
end;

procedure TRESTClient.SetBasicAuth(const Username, Password: string);
begin
  FHTTP.Request.BasicAuthentication := True;
  FHTTP.Request.Username := Username;
  FHTTP.Request.Password := Password;
end;

procedure TRESTClient.SetBearerToken(const Token: string);
begin
  FAuthToken := Token;
  FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + Token;
end;

function TRESTClient.ResponseToJSON(const Response: string): TJSONData;
begin
  try
    Result := GetJSON(Response);
  except
    Result := nil;
  end;
end;

function TRESTClient.GetString(const Path: string): string;
begin
  try
    Result := FHTTP.Get(BuildURL(Path));
  except
    on E: EIdHTTPProtocolException do
    begin
      Result := E.ErrorMessage;
      raise;
    end;
  end;
end;

function TRESTClient.Get(const Path: string): TJSONData;
begin
  Result := ResponseToJSON(GetString(Path));
end;

function TRESTClient.PostString(const Path, Body, ContentType: string): string;
var
  Stream: TStringStream;
begin
  Stream := TStringStream.Create(Body, TEncoding.UTF8);
  try
    FHTTP.Request.ContentType := ContentType;
    Result := FHTTP.Post(BuildURL(Path), Stream);
  finally
    Stream.Free;
  end;
end;

function TRESTClient.Post(const Path: string; const Body: TJSONObject): TJSONData;
var
  JSONStr: string;
begin
  JSONStr := Body.AsJSON;
  Result := ResponseToJSON(PostString(Path, JSONStr, 'application/json'));
end;

function TRESTClient.Put(const Path: string; const Body: TJSONObject): TJSONData;
var
  Stream: TStringStream;
  Response: string;
begin
  Stream := TStringStream.Create(Body.AsJSON, TEncoding.UTF8);
  try
    FHTTP.Request.ContentType := 'application/json';
    Response := FHTTP.Put(BuildURL(Path), Stream);
    Result := ResponseToJSON(Response);
  finally
    Stream.Free;
  end;
end;

function TRESTClient.Patch(const Path: string; const Body: TJSONObject): TJSONData;
var
  Stream: TStringStream;
  Response: string;
begin
  Stream := TStringStream.Create(Body.AsJSON, TEncoding.UTF8);
  try
    FHTTP.Request.ContentType := 'application/json';
    Response := FHTTP.Patch(BuildURL(Path), Stream);
    Result := ResponseToJSON(Response);
  finally
    Stream.Free;
  end;
end;

function TRESTClient.Delete(const Path: string): Boolean;
begin
  try
    FHTTP.Delete(BuildURL(Path));
    Result := FHTTP.ResponseCode in [200, 204];
  except
    Result := False;
  end;
end;

function TRESTClient.UploadFile(const Path, FieldName, FileName: string): TJSONData;
var
  FormData: TIdMultiPartFormDataStream;
  Response: string;
begin
  FormData := TIdMultiPartFormDataStream.Create;
  try
    FormData.AddFile(FieldName, FileName);
    Response := FHTTP.Post(BuildURL(Path), FormData);
    Result := ResponseToJSON(Response);
  finally
    FormData.Free;
  end;
end;

end.
```

---

## 40.5 Request Headers

```pascal
unit HTTPHeaders;

// การจัดการ HTTP Headers

procedure SetCustomHeaders(AHTTP: TIdHTTP);
begin
  // Standard Headers
  AHTTP.Request.UserAgent := 'MyApp/1.0 (Windows; Pascal)';
  AHTTP.Request.Accept := 'application/json';
  AHTTP.Request.AcceptLanguage := 'th-TH,en;q=0.9';
  AHTTP.Request.AcceptEncoding := 'gzip, deflate';
  AHTTP.Request.Connection := 'Keep-Alive';
  
  // Custom Headers
  AHTTP.Request.CustomHeaders.Values['X-API-Key'] := 'your-api-key-here';
  AHTTP.Request.CustomHeaders.Values['X-Client-ID'] := 'client-123';
  AHTTP.Request.CustomHeaders.Values['Cache-Control'] := 'no-cache';
  AHTTP.Request.CustomHeaders.Values['X-Request-ID'] := 
    LowerCase(GUIDToString(TGuid.NewGuid));
end;

// อ่าน Response Headers
procedure ReadResponseHeaders(AHTTP: TIdHTTP);
var
  i: Integer;
begin
  WriteLn('Status: ', AHTTP.ResponseText);
  WriteLn('Content-Type: ', AHTTP.Response.ContentType);
  WriteLn('Content-Length: ', AHTTP.Response.ContentLength);
  WriteLn('Last-Modified: ', AHTTP.Response.LastModified);
  WriteLn('ETag: ', AHTTP.Response.ETag);
  WriteLn('Server: ', AHTTP.Response.Server);
  
  // ทุก Headers
  for i := 0 to AHTTP.Response.RawHeaders.Count - 1 do
    WriteLn(AHTTP.Response.RawHeaders[i]);
end;
```

---

## 40.6 Authentication

```pascal
unit HTTPAuth;

interface

uses
  Classes, SysUtils, IdHTTP, IdSSLOpenSSL, IdAuthentication,
  SynCrypto, SynCommons;  // สำหรับ HMAC SHA-256

type
  TAuthType = (atNone, atBasic, atBearer, atDigest, atApiKey, atOAuth2);
  
  THTTPAuth = class
  private
    FHTTP: TIdHTTP;
  public
    constructor Create(AHTTP: TIdHTTP);
    
    procedure SetBasicAuth(const Username, Password: string);
    procedure SetBearerToken(const Token: string);
    procedure SetApiKey(const KeyName, KeyValue: string; 
                         InHeader: Boolean = True);
    procedure SetOAuth2(const AccessToken: string);
    procedure SetHMACAuth(const APIKey, SecretKey: string);
    procedure ClearAuth;
  end;

implementation

uses
  EncdDecd;

constructor THTTPAuth.Create(AHTTP: TIdHTTP);
begin
  inherited Create;
  FHTTP := AHTTP;
end;

procedure THTTPAuth.SetBasicAuth(const Username, Password: string);
var
  Encoded: string;
begin
  // Basic Auth: Base64(username:password)
  Encoded := EncodeStringBase64(Username + ':' + Password);
  FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Basic ' + Encoded;
end;

procedure THTTPAuth.SetBearerToken(const Token: string);
begin
  FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + Token;
end;

procedure THTTPAuth.SetApiKey(const KeyName, KeyValue: string; InHeader: Boolean);
begin
  if InHeader then
    FHTTP.Request.CustomHeaders.Values[KeyName] := KeyValue
  else
  begin
    // ใส่ใน Query String แทน (ต้องจัดการ URL เอง)
    // FHTTP.Get(URL + '?' + KeyName + '=' + KeyValue);
  end;
end;

procedure THTTPAuth.SetOAuth2(const AccessToken: string);
begin
  FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + AccessToken;
end;

procedure THTTPAuth.SetHMACAuth(const APIKey, SecretKey: string);
var
  Timestamp: string;
  Nonce: string;
  StringToSign: string;
  Signature: string;
begin
  // สร้าง HMAC-SHA256 Signature (ตัวอย่าง AWS-style)
  Timestamp := FormatDateTime('yyyymmddThhnnss', TTimeZone.Local.ToUniversalTime(Now)) + 'Z';
  Nonce := LowerCase(GUIDToString(TGuid.NewGuid)).Replace('-', '').Replace('{', '').Replace('}', '');
  
  StringToSign := APIKey + #10 + Timestamp + #10 + Nonce;
  
  // HMAC-SHA256 (ต้องใช้ library เช่น OpenSSL หรือ DCPcrypt)
  // Signature := HMACSHA256(SecretKey, StringToSign);
  
  FHTTP.Request.CustomHeaders.Values['X-API-Key'] := APIKey;
  FHTTP.Request.CustomHeaders.Values['X-Timestamp'] := Timestamp;
  FHTTP.Request.CustomHeaders.Values['X-Nonce'] := Nonce;
  // FHTTP.Request.CustomHeaders.Values['X-Signature'] := Signature;
end;

procedure THTTPAuth.ClearAuth;
begin
  FHTTP.Request.CustomHeaders.Values['Authorization'] := '';
  FHTTP.Request.CustomHeaders.Values['X-API-Key'] := '';
  FHTTP.Request.BasicAuthentication := False;
end;

end.
```

---

## 40.7 File Upload (Multipart)

```pascal
unit FileUpload;

interface

uses
  Classes, SysUtils,
  IdHTTP, IdSSLOpenSSL, IdMultipartFormData;

type
  TFileUploader = class
  private
    FHTTP: TIdHTTP;
    FSSLHandler: TIdSSLIOHandlerSocketOpenSSL;
    FOnProgress: procedure(Sent, Total: Int64) of object;
  public
    constructor Create;
    destructor Destroy; override;
    
    function UploadSingleFile(const URL, FieldName, FileName: string): string;
    function UploadMultipleFiles(const URL: string; 
                                  const Fields: array of string;
                                  const Files: array of string): string;
    function UploadWithFormData(const URL: string;
                                 const FormFields: TStringList;
                                 const FileFields: TStringList): string;
    
    property OnProgress: procedure(Sent, Total: Int64) of object 
             read FOnProgress write FOnProgress;
  end;

implementation

constructor TFileUploader.Create;
begin
  inherited;
  FHTTP := TIdHTTP.Create(nil);
  FHTTP.ConnectTimeout := 30000;
  FHTTP.ReadTimeout := 300000;  // 5 minutes for upload
  
  FSSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  FSSLHandler.SSLOptions.Method := sslvSSLv23;
  FHTTP.IOHandler := FSSLHandler;
  
  FHTTP.Request.UserAgent := 'Lazarus File Uploader 1.0';
end;

destructor TFileUploader.Destroy;
begin
  FHTTP.Free;
  FSSLHandler.Free;
  inherited;
end;

function TFileUploader.UploadSingleFile(const URL, FieldName, FileName: string): string;
var
  FormData: TIdMultiPartFormDataStream;
begin
  FormData := TIdMultiPartFormDataStream.Create;
  try
    // เพิ่มไฟล์เป็น Multipart Field
    FormData.AddFile(FieldName, FileName);
    
    // ตั้งค่า Content-Type อัตโนมัติ
    FHTTP.Request.ContentType := FormData.RequestContentType;
    
    Result := FHTTP.Post(URL, FormData);
  finally
    FormData.Free;
  end;
end;

function TFileUploader.UploadMultipleFiles(const URL: string;
  const Fields: array of string; const Files: array of string): string;
var
  FormData: TIdMultiPartFormDataStream;
  i: Integer;
begin
  if Length(Fields) <> Length(Files) then
    raise Exception.Create('Fields and Files arrays must have same length');
  
  FormData := TIdMultiPartFormDataStream.Create;
  try
    for i := 0 to High(Files) do
      FormData.AddFile(Fields[i], Files[i]);
    
    FHTTP.Request.ContentType := FormData.RequestContentType;
    Result := FHTTP.Post(URL, FormData);
  finally
    FormData.Free;
  end;
end;

function TFileUploader.UploadWithFormData(const URL: string;
  const FormFields: TStringList; const FileFields: TStringList): string;
var
  FormData: TIdMultiPartFormDataStream;
  i: Integer;
begin
  FormData := TIdMultiPartFormDataStream.Create;
  try
    // เพิ่ม Text Fields
    for i := 0 to FormFields.Count - 1 do
      FormData.AddFormField(FormFields.Names[i], FormFields.ValueFromIndex[i]);
    
    // เพิ่ม File Fields (Format: "fieldname=filepath")
    for i := 0 to FileFields.Count - 1 do
      FormData.AddFile(FileFields.Names[i], FileFields.ValueFromIndex[i]);
    
    FHTTP.Request.ContentType := FormData.RequestContentType;
    Result := FHTTP.Post(URL, FormData);
  finally
    FormData.Free;
  end;
end;

// ตัวอย่างการใช้งาน
procedure DemoUpload;
var
  Uploader: TFileUploader;
  FormFields, FileFields: TStringList;
begin
  Uploader := TFileUploader.Create;
  FormFields := TStringList.Create;
  FileFields := TStringList.Create;
  try
    // ส่งข้อมูล Form + ไฟล์
    FormFields.Add('username=john_doe');
    FormFields.Add('description=My Photo');
    FormFields.Add('category=profile');
    
    FileFields.Add('avatar=' + ExtractFilePath(Application.ExeName) + 'photo.jpg');
    FileFields.Add('document=C:\Files\doc.pdf');
    
    var Response := Uploader.UploadWithFormData(
      'https://api.example.com/upload',
      FormFields, FileFields
    );
    WriteLn('Upload result: ', Response);
    
  finally
    Uploader.Free;
    FormFields.Free;
    FileFields.Free;
  end;
end;

end.
```

---

## 40.8 Cookie Management

```pascal
unit CookieManager;

interface

uses
  Classes, SysUtils, IdHTTP, IdCookieManager, IdCookie;

type
  TCookieHelper = class
  private
    FHTTP: TIdHTTP;
    FCookieManager: TIdCookieManager;
  public
    constructor Create(AHTTP: TIdHTTP);
    destructor Destroy; override;
    
    procedure SaveCookies(const FileName: string);
    procedure LoadCookies(const FileName: string);
    procedure ClearCookies;
    function GetCookieValue(const Name: string): string;
    function GetCookieCount: Integer;
    procedure ListCookies(AList: TStringList);
    procedure SetCookie(const Name, Value, Domain: string; 
                         ExpiresInDays: Integer = 30);
  end;

implementation

constructor TCookieHelper.Create(AHTTP: TIdHTTP);
begin
  inherited Create;
  FHTTP := AHTTP;
  
  // สร้างและ Attach Cookie Manager
  FCookieManager := TIdCookieManager.Create(nil);
  FHTTP.CookieManager := FCookieManager;
  FHTTP.AllowCookies := True;
  FHTTP.HandleRedirects := True;
end;

destructor TCookieHelper.Destroy;
begin
  FCookieManager.Free;
  inherited;
end;

procedure TCookieHelper.SaveCookies(const FileName: string);
var
  SL: TStringList;
  i: Integer;
  Cookie: TIdCookie;
begin
  SL := TStringList.Create;
  try
    for i := 0 to FCookieManager.CookieCollection.Count - 1 do
    begin
      Cookie := FCookieManager.CookieCollection.Cookies[i];
      SL.Add(Format('%s=%s|%s|%s|%s', [
        Cookie.CookieName,
        Cookie.Value,
        Cookie.Domain,
        Cookie.Path,
        DateTimeToStr(Cookie.Expires)
      ]));
    end;
    SL.SaveToFile(FileName);
  finally
    SL.Free;
  end;
end;

procedure TCookieHelper.LoadCookies(const FileName: string);
var
  SL: TStringList;
  i: Integer;
  Parts: TStringArray;
  Cookie: TIdCookie;
begin
  if not FileExists(FileName) then Exit;
  
  SL := TStringList.Create;
  try
    SL.LoadFromFile(FileName);
    for i := 0 to SL.Count - 1 do
    begin
      Parts := SL[i].Split(['|']);
      if Length(Parts) >= 4 then
      begin
        Cookie := TIdCookie.Create(FCookieManager.CookieCollection);
        var EqPos := Pos('=', Parts[0]);
        if EqPos > 0 then
        begin
          Cookie.CookieName := Copy(Parts[0], 1, EqPos - 1);
          Cookie.Value := Copy(Parts[0], EqPos + 1, MaxInt);
        end;
        Cookie.Domain := Parts[1];
        Cookie.Path := Parts[2];
        if Length(Parts) >= 5 then
          Cookie.Expires := StrToDateTimeDef(Parts[4], 0);
      end;
    end;
  finally
    SL.Free;
  end;
end;

procedure TCookieHelper.ClearCookies;
begin
  FCookieManager.CookieCollection.Clear;
end;

function TCookieHelper.GetCookieValue(const Name: string): string;
var
  i: Integer;
begin
  Result := '';
  for i := 0 to FCookieManager.CookieCollection.Count - 1 do
  begin
    if FCookieManager.CookieCollection.Cookies[i].CookieName = Name then
    begin
      Result := FCookieManager.CookieCollection.Cookies[i].Value;
      Break;
    end;
  end;
end;

function TCookieHelper.GetCookieCount: Integer;
begin
  Result := FCookieManager.CookieCollection.Count;
end;

procedure TCookieHelper.ListCookies(AList: TStringList);
var
  i: Integer;
  Cookie: TIdCookie;
begin
  AList.Clear;
  for i := 0 to FCookieManager.CookieCollection.Count - 1 do
  begin
    Cookie := FCookieManager.CookieCollection.Cookies[i];
    AList.Add(Format('Name: %s, Value: %s, Domain: %s, Expires: %s',
      [Cookie.CookieName, Cookie.Value, Cookie.Domain, 
       DateTimeToStr(Cookie.Expires)]));
  end;
end;

procedure TCookieHelper.SetCookie(const Name, Value, Domain: string;
  ExpiresInDays: Integer);
var
  Cookie: TIdCookie;
begin
  Cookie := TIdCookie.Create(FCookieManager.CookieCollection);
  Cookie.CookieName := Name;
  Cookie.Value := Value;
  Cookie.Domain := Domain;
  Cookie.Path := '/';
  Cookie.Expires := Now + ExpiresInDays;
  Cookie.Secure := True;
  Cookie.HttpOnly := True;
end;

end.
```

---

## 40.9 ตัวอย่างสมบูรณ์: Weather App ด้วย OpenWeatherMap API

```pascal
unit WeatherApp;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls, Graphics,
  IdHTTP, IdSSLOpenSSL,
  fpjson, jsonparser;

const
  OPENWEATHER_API_KEY = 'YOUR_API_KEY_HERE';
  OPENWEATHER_BASE_URL = 'https://api.openweathermap.org/data/2.5/';

type
  TWeatherData = record
    City: string;
    Country: string;
    Temperature: Double;
    FeelsLike: Double;
    MinTemp: Double;
    MaxTemp: Double;
    Humidity: Integer;
    Pressure: Integer;
    WindSpeed: Double;
    WindDeg: Integer;
    Description: string;
    Icon: string;
    Visibility: Integer;
    Clouds: Integer;
    Sunrise: TDateTime;
    Sunset: TDateTime;
  end;
  
  TForecastItem = record
    DateTime: TDateTime;
    Temperature: Double;
    Description: string;
    Icon: string;
    Humidity: Integer;
    WindSpeed: Double;
  end;
  
  TForm1 = class(TForm)
    EditCity: TEdit;
    EditCountry: TEdit;
    btnSearch: TButton;
    btnRefresh: TButton;
    
    PanelCurrent: TPanel;
    lblCityName: TLabel;
    lblTemp: TLabel;
    lblFeelsLike: TLabel;
    lblMinMax: TLabel;
    lblDescription: TLabel;
    lblHumidity: TLabel;
    lblWind: TLabel;
    lblPressure: TLabel;
    lblVisibility: TLabel;
    lblSunrise: TLabel;
    lblSunset: TLabel;
    
    PanelForecast: TPanel;
    ListViewForecast: TListView;
    
    StatusBar1: TStatusBar;
    
    IdHTTP1: TIdHTTP;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnSearchClick(Sender: TObject);
    procedure btnRefreshClick(Sender: TObject);
  private
    FSSLHandler: TIdSSLIOHandlerSocketOpenSSL;
    FLastCity: string;
    FLastCountry: string;
    FCurrentWeather: TWeatherData;
    
    function GetCurrentWeather(const City, CountryCode: string): Boolean;
    function Get5DayForecast(const City, CountryCode: string): Boolean;
    procedure UpdateWeatherDisplay;
    procedure UpdateForecastDisplay(const Items: array of TForecastItem);
    function UnixToDateTime(UnixTime: Int64): TDateTime;
    function DegreesToDirection(Degrees: Integer): string;
    function GetWindDescription(Speed: Double): string;
    function ParseWeatherResponse(const JSON: string; 
                                    out Data: TWeatherData): Boolean;
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  FSSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  FSSLHandler.SSLOptions.Method := sslvSSLv23;
  FSSLHandler.SSLOptions.Mode := sslmClient;
  IdHTTP1.IOHandler := FSSLHandler;
  IdHTTP1.ConnectTimeout := 10000;
  IdHTTP1.ReadTimeout := 15000;
  IdHTTP1.Request.UserAgent := 'Lazarus Weather App 1.0';
  
  FLastCity := 'Bangkok';
  FLastCountry := 'TH';
  
  EditCity.Text := FLastCity;
  EditCountry.Text := FLastCountry;
  
  // ตั้งค่า Forecast ListView
  with ListViewForecast.Columns do
  begin
    with Add do begin Caption := 'วันที่'; Width := 100; end;
    with Add do begin Caption := 'เวลา'; Width := 70; end;
    with Add do begin Caption := 'อุณหภูมิ (°C)'; Width := 100; end;
    with Add do begin Caption := 'สภาพอากาศ'; Width := 150; end;
    with Add do begin Caption := 'ความชื้น'; Width := 80; end;
    with Add do begin Caption := 'ลม (km/h)'; Width := 80; end;
  end;
  ListViewForecast.ViewStyle := vsReport;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FSSLHandler.Free;
end;

function TForm1.UnixToDateTime(UnixTime: Int64): TDateTime;
const
  UnixStartDate = 25569.0;  // 1/1/1970
begin
  Result := UnixStartDate + (UnixTime / 86400);
end;

function TForm1.DegreesToDirection(Degrees: Integer): string;
const
  Directions: array[0..7] of string = ('N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW');
begin
  Result := Directions[Round(Degrees / 45) mod 8];
end;

function TForm1.GetWindDescription(Speed: Double): string;
begin
  if Speed < 1 then Result := 'สงบ'
  else if Speed < 6 then Result := 'ลมเบา'
  else if Speed < 12 then Result := 'ลมอ่อน'
  else if Speed < 20 then Result := 'ลมปานกลาง'
  else if Speed < 29 then Result := 'ลมแรง'
  else if Speed < 39 then Result := 'ลมแรงมาก'
  else Result := 'พายุ';
end;

function TForm1.ParseWeatherResponse(const JSON: string; 
  out Data: TWeatherData): Boolean;
var
  JData, JMain, JWeather, JWind, JSys: TJSONData;
  JObj: TJSONObject;
  JArr: TJSONArray;
begin
  Result := False;
  FillChar(Data, SizeOf(Data), 0);
  
  try
    JData := GetJSON(JSON);
    if not (JData is TJSONObject) then Exit;
    JObj := TJSONObject(JData);
    
    // ชื่อเมือง
    Data.City := JObj.Get('name', '');
    
    // ข้อมูล sys (country, sunrise, sunset)
    JSys := JObj.Find('sys');
    if JSys is TJSONObject then
    begin
      Data.Country := TJSONObject(JSys).Get('country', '');
      Data.Sunrise := UnixToDateTime(TJSONObject(JSys).Get('sunrise', Int64(0)));
      Data.Sunset := UnixToDateTime(TJSONObject(JSys).Get('sunset', Int64(0)));
    end;
    
    // ข้อมูลอุณหภูมิ
    JMain := JObj.Find('main');
    if JMain is TJSONObject then
    with TJSONObject(JMain) do
    begin
      Data.Temperature := Get('temp', 0.0) - 273.15;  // Kelvin to Celsius
      Data.FeelsLike := Get('feels_like', 0.0) - 273.15;
      Data.MinTemp := Get('temp_min', 0.0) - 273.15;
      Data.MaxTemp := Get('temp_max', 0.0) - 273.15;
      Data.Humidity := Get('humidity', 0);
      Data.Pressure := Get('pressure', 0);
    end;
    
    // สภาพอากาศ
    JWeather := JObj.Find('weather');
    if (JWeather is TJSONArray) and (TJSONArray(JWeather).Count > 0) then
    begin
      var WItem := TJSONArray(JWeather).Items[0];
      if WItem is TJSONObject then
      with TJSONObject(WItem) do
      begin
        Data.Description := Get('description', '');
        Data.Icon := Get('icon', '');
      end;
    end;
    
    // ลม
    JWind := JObj.Find('wind');
    if JWind is TJSONObject then
    with TJSONObject(JWind) do
    begin
      Data.WindSpeed := Get('speed', 0.0) * 3.6;  // m/s to km/h
      Data.WindDeg := Get('deg', 0);
    end;
    
    // Visibility
    Data.Visibility := JObj.Get('visibility', 0);
    
    // Cloud cover
    var JClouds := JObj.Find('clouds');
    if JClouds is TJSONObject then
      Data.Clouds := TJSONObject(JClouds).Get('all', 0);
    
    Result := True;
    
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Parse error: ' + E.Message;
  end;
end;

function TForm1.GetCurrentWeather(const City, CountryCode: string): Boolean;
var
  URL, Response: string;
begin
  Result := False;
  
  URL := Format('%sweather?q=%s,%s&appid=%s&lang=th',
    [OPENWEATHER_BASE_URL, City, CountryCode, OPENWEATHER_API_KEY]);
  
  try
    Response := IdHTTP1.Get(URL);
    Result := ParseWeatherResponse(Response, FCurrentWeather);
    
    if Result then
      UpdateWeatherDisplay;
      
  except
    on E: EIdHTTPProtocolException do
    begin
      if E.ErrorCode = 404 then
        StatusBar1.SimpleText := 'ไม่พบเมืองที่ระบุ'
      else if E.ErrorCode = 401 then
        StatusBar1.SimpleText := 'API Key ไม่ถูกต้อง'
      else
        StatusBar1.SimpleText := 'HTTP Error ' + IntToStr(E.ErrorCode);
    end;
    on E: Exception do
      StatusBar1.SimpleText := 'Error: ' + E.Message;
  end;
end;

function TForm1.Get5DayForecast(const City, CountryCode: string): Boolean;
var
  URL, Response: string;
  JData, JList, JItem: TJSONData;
  Items: array of TForecastItem;
  i: Integer;
begin
  Result := False;
  
  URL := Format('%sforecast?q=%s,%s&appid=%s&lang=th',
    [OPENWEATHER_BASE_URL, City, CountryCode, OPENWEATHER_API_KEY]);
  
  try
    Response := IdHTTP1.Get(URL);
    
    JData := GetJSON(Response);
    if not (JData is TJSONObject) then Exit;
    
    JList := TJSONObject(JData).Find('list');
    if not (JList is TJSONArray) then Exit;
    
    SetLength(Items, TJSONArray(JList).Count);
    
    for i := 0 to TJSONArray(JList).Count - 1 do
    begin
      JItem := TJSONArray(JList).Items[i];
      if not (JItem is TJSONObject) then Continue;
      
      var JObj := TJSONObject(JItem);
      
      Items[i].DateTime := UnixToDateTime(JObj.Get('dt', Int64(0)));
      
      var JMain := JObj.Find('main');
      if JMain is TJSONObject then
        Items[i].Temperature := TJSONObject(JMain).Get('temp', 0.0) - 273.15;
      
      var JWeather := JObj.Find('weather');
      if (JWeather is TJSONArray) and (TJSONArray(JWeather).Count > 0) then
      begin
        var WItem := TJSONArray(JWeather).Items[0];
        if WItem is TJSONObject then
          Items[i].Description := TJSONObject(WItem).Get('description', '');
      end;
      
      if JMain is TJSONObject then
        Items[i].Humidity := TJSONObject(JMain).Get('humidity', 0);
      
      var JWind := JObj.Find('wind');
      if JWind is TJSONObject then
        Items[i].WindSpeed := TJSONObject(JWind).Get('speed', 0.0) * 3.6;
    end;
    
    UpdateForecastDisplay(Items);
    Result := True;
    
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Forecast error: ' + E.Message;
  end;
end;

procedure TForm1.UpdateWeatherDisplay;
var
  D: TWeatherData;
begin
  D := FCurrentWeather;
  
  // แสดงข้อมูลปัจจุบัน
  lblCityName.Caption := Format('%s, %s', [D.City, D.Country]);
  lblTemp.Caption := Format('%.1f°C', [D.Temperature]);
  lblFeelsLike.Caption := Format('รู้สึกเหมือน %.1f°C', [D.FeelsLike]);
  lblMinMax.Caption := Format('ต่ำสุด: %.1f°C | สูงสุด: %.1f°C', [D.MinTemp, D.MaxTemp]);
  lblDescription.Caption := D.Description;
  lblHumidity.Caption := Format('ความชื้น: %d%%', [D.Humidity]);
  lblWind.Caption := Format('ลม: %.1f km/h %s (%s)', 
    [D.WindSpeed, DegreesToDirection(D.WindDeg), GetWindDescription(D.WindSpeed)]);
  lblPressure.Caption := Format('ความกดอากาศ: %d hPa', [D.Pressure]);
  lblVisibility.Caption := Format('ทัศนวิสัย: %.1f km', [D.Visibility / 1000]);
  lblSunrise.Caption := Format('พระอาทิตย์ขึ้น: %s', [FormatDateTime('hh:nn', D.Sunrise)]);
  lblSunset.Caption := Format('พระอาทิตย์ตก: %s', [FormatDateTime('hh:nn', D.Sunset)]);
  
  StatusBar1.SimpleText := 'อัปเดตล่าสุด: ' + FormatDateTime('dd/mm/yyyy hh:nn:ss', Now);
end;

procedure TForm1.UpdateForecastDisplay(const Items: array of TForecastItem);
var
  i: Integer;
  Item: TListItem;
begin
  ListViewForecast.Clear;
  
  for i := 0 to High(Items) do
  begin
    Item := ListViewForecast.Items.Add;
    Item.Caption := FormatDateTime('ddd dd/mm', Items[i].DateTime);
    Item.SubItems.Add(FormatDateTime('hh:nn', Items[i].DateTime));
    Item.SubItems.Add(Format('%.1f°C', [Items[i].Temperature]));
    Item.SubItems.Add(Items[i].Description);
    Item.SubItems.Add(Format('%d%%', [Items[i].Humidity]));
    Item.SubItems.Add(Format('%.1f', [Items[i].WindSpeed]));
  end;
end;

procedure TForm1.btnSearchClick(Sender: TObject);
var
  City, Country: string;
begin
  City := Trim(EditCity.Text);
  Country := Trim(EditCountry.Text);
  
  if City = '' then
  begin
    ShowMessage('กรุณาใส่ชื่อเมือง');
    Exit;
  end;
  
  FLastCity := City;
  FLastCountry := Country;
  
  StatusBar1.SimpleText := 'กำลังโหลดข้อมูล...';
  Application.ProcessMessages;
  
  btnSearch.Enabled := False;
  try
    GetCurrentWeather(City, Country);
    Get5DayForecast(City, Country);
  finally
    btnSearch.Enabled := True;
  end;
end;

procedure TForm1.btnRefreshClick(Sender: TObject);
begin
  if FLastCity <> '' then
  begin
    EditCity.Text := FLastCity;
    EditCountry.Text := FLastCountry;
    btnSearchClick(nil);
  end;
end;

end.
```

---

## 40.10 ตัวอย่างสมบูรณ์: GitHub API Client

```pascal
unit GitHubClient;

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls, ComCtrls,
  IdHTTP, IdSSLOpenSSL,
  fpjson, jsonparser;

const
  GITHUB_API_BASE = 'https://api.github.com';
  GITHUB_API_VERSION = '2022-11-28';

type
  TGitHubRepo = record
    ID: Integer;
    Name: string;
    FullName: string;
    Description: string;
    URL: string;
    Stars: Integer;
    Forks: Integer;
    OpenIssues: Integer;
    Language: string;
    Private: Boolean;
    CreatedAt: string;
    UpdatedAt: string;
    DefaultBranch: string;
    Size: Integer;
  end;
  
  TGitHubUser = record
    Login: string;
    ID: Integer;
    Name: string;
    Email: string;
    Bio: string;
    Company: string;
    Location: string;
    Blog: string;
    PublicRepos: Integer;
    Followers: Integer;
    Following: Integer;
    CreatedAt: string;
  end;
  
  TGitHubIssue = record
    Number: Integer;
    Title: string;
    State: string;
    Body: string;
    Author: string;
    CreatedAt: string;
    Labels: TStringList;
  end;
  
  TForm1 = class(TForm)
    EditToken: TEdit;
    EditUsername: TEdit;
    EditRepo: TEdit;
    
    TabControl1: TPageControl;
    TabUser: TTabSheet;
    TabRepos: TTabSheet;
    TabIssues: TTabSheet;
    
    // User Info Panel
    lblUserLogin: TLabel;
    lblUserName: TLabel;
    lblUserBio: TLabel;
    lblUserLocation: TLabel;
    lblUserRepos: TLabel;
    lblUserFollowers: TLabel;
    
    // Repos List
    ListViewRepos: TListView;
    btnListRepos: TButton;
    btnCreateRepo: TButton;
    
    // Issues
    ListViewIssues: TListView;
    btnListIssues: TButton;
    btnCreateIssue: TButton;
    
    btnLoadUser: TButton;
    StatusBar1: TStatusBar;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnLoadUserClick(Sender: TObject);
    procedure btnListReposClick(Sender: TObject);
    procedure btnCreateRepoClick(Sender: TObject);
    procedure btnListIssuesClick(Sender: TObject);
    procedure btnCreateIssueClick(Sender: TObject);
  private
    FHTTP: TIdHTTP;
    FSSLHandler: TIdSSLIOHandlerSocketOpenSSL;
    FToken: string;
    
    procedure SetupHTTP;
    function GetRequest(const Path: string): string;
    function PostRequest(const Path, Body: string): string;
    function PatchRequest(const Path, Body: string): string;
    function DeleteRequest(const Path: string): Boolean;
    
    function GetUserInfo(const Username: string): TGitHubUser;
    function GetRepos(const Username: string): TArray<TGitHubRepo>;
    function GetIssues(const Owner, Repo: string): TArray<TGitHubIssue>;
    function CreateRepo(const Name, Description: string; 
                         Private_: Boolean): TGitHubRepo;
    function CreateIssue(const Owner, Repo, Title, Body: string): TGitHubIssue;
    
    procedure DisplayUser(const User: TGitHubUser);
    procedure DisplayRepos(const Repos: TArray<TGitHubRepo>);
    procedure DisplayIssues(const Issues: TArray<TGitHubIssue>);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.FormCreate(Sender: TObject);
begin
  SetupHTTP;
  
  // ตั้งค่า Repos ListView
  with ListViewRepos.Columns do
  begin
    with Add do begin Caption := 'Repository'; Width := 200; end;
    with Add do begin Caption := 'Language'; Width := 100; end;
    with Add do begin Caption := '⭐ Stars'; Width := 80; end;
    with Add do begin Caption := '🍴 Forks'; Width := 80; end;
    with Add do begin Caption := '🔴 Issues'; Width := 80; end;
    with Add do begin Caption := 'Updated'; Width := 120; end;
  end;
  ListViewRepos.ViewStyle := vsReport;
  
  // ตั้งค่า Issues ListView
  with ListViewIssues.Columns do
  begin
    with Add do begin Caption := '#'; Width := 60; end;
    with Add do begin Caption := 'Title'; Width := 300; end;
    with Add do begin Caption := 'State'; Width := 80; end;
    with Add do begin Caption := 'Author'; Width := 120; end;
    with Add do begin Caption := 'Created'; Width := 120; end;
  end;
  ListViewIssues.ViewStyle := vsReport;
end;

procedure TForm1.FormDestroy(Sender: TObject);
begin
  FHTTP.Free;
  FSSLHandler.Free;
end;

procedure TForm1.SetupHTTP;
begin
  FHTTP := TIdHTTP.Create(nil);
  FHTTP.ConnectTimeout := 10000;
  FHTTP.ReadTimeout := 30000;
  FHTTP.HandleRedirects := True;
  
  FSSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(nil);
  FSSLHandler.SSLOptions.Method := sslvSSLv23;
  FHTTP.IOHandler := FSSLHandler;
  
  // GitHub API Headers
  FHTTP.Request.UserAgent := 'Lazarus GitHub Client 1.0';
  FHTTP.Request.Accept := 'application/vnd.github+json';
  FHTTP.Request.CustomHeaders.Values['X-GitHub-Api-Version'] := GITHUB_API_VERSION;
end;

function TForm1.GetRequest(const Path: string): string;
begin
  FToken := Trim(EditToken.Text);
  if FToken <> '' then
    FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + FToken;
  
  try
    Result := FHTTP.Get(GITHUB_API_BASE + Path);
  except
    on E: EIdHTTPProtocolException do
    begin
      StatusBar1.SimpleText := Format('API Error %d: %s', [E.ErrorCode, E.Message]);
      Result := '';
    end;
    on E: Exception do
    begin
      StatusBar1.SimpleText := 'Error: ' + E.Message;
      Result := '';
    end;
  end;
end;

function TForm1.PostRequest(const Path, Body: string): string;
var
  Stream: TStringStream;
begin
  FToken := Trim(EditToken.Text);
  if FToken <> '' then
    FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + FToken;
  
  FHTTP.Request.ContentType := 'application/json';
  Stream := TStringStream.Create(Body, TEncoding.UTF8);
  try
    Result := FHTTP.Post(GITHUB_API_BASE + Path, Stream);
  except
    on E: Exception do
    begin
      StatusBar1.SimpleText := 'POST Error: ' + E.Message;
      Result := '';
    end;
  finally
    Stream.Free;
  end;
end;

function TForm1.PatchRequest(const Path, Body: string): string;
var
  Stream: TStringStream;
begin
  FToken := Trim(EditToken.Text);
  if FToken <> '' then
    FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + FToken;
  
  FHTTP.Request.ContentType := 'application/json';
  Stream := TStringStream.Create(Body, TEncoding.UTF8);
  try
    Result := FHTTP.Patch(GITHUB_API_BASE + Path, Stream);
  except
    on E: Exception do
    begin
      Result := '';
    end;
  finally
    Stream.Free;
  end;
end;

function TForm1.DeleteRequest(const Path: string): Boolean;
begin
  FToken := Trim(EditToken.Text);
  if FToken <> '' then
    FHTTP.Request.CustomHeaders.Values['Authorization'] := 'Bearer ' + FToken;
  
  try
    FHTTP.Delete(GITHUB_API_BASE + Path);
    Result := FHTTP.ResponseCode in [200, 204];
  except
    Result := False;
  end;
end;

function TForm1.GetUserInfo(const Username: string): TGitHubUser;
var
  Response: string;
  JObj: TJSONObject;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  Response := GetRequest('/users/' + Username);
  if Response = '' then Exit;
  
  try
    var JData := GetJSON(Response);
    if not (JData is TJSONObject) then Exit;
    JObj := TJSONObject(JData);
    
    Result.Login := JObj.Get('login', '');
    Result.ID := JObj.Get('id', 0);
    Result.Name := JObj.Get('name', '');
    Result.Email := JObj.Get('email', '');
    Result.Bio := JObj.Get('bio', '');
    Result.Company := JObj.Get('company', '');
    Result.Location := JObj.Get('location', '');
    Result.Blog := JObj.Get('blog', '');
    Result.PublicRepos := JObj.Get('public_repos', 0);
    Result.Followers := JObj.Get('followers', 0);
    Result.Following := JObj.Get('following', 0);
    Result.CreatedAt := JObj.Get('created_at', '');
    
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Parse error: ' + E.Message;
  end;
end;

function TForm1.GetRepos(const Username: string): TArray<TGitHubRepo>;
var
  Response: string;
  JArr: TJSONArray;
  i: Integer;
begin
  SetLength(Result, 0);
  
  Response := GetRequest(Format('/users/%s/repos?per_page=100&sort=updated', [Username]));
  if Response = '' then Exit;
  
  try
    var JData := GetJSON(Response);
    if not (JData is TJSONArray) then Exit;
    JArr := TJSONArray(JData);
    
    SetLength(Result, JArr.Count);
    for i := 0 to JArr.Count - 1 do
    begin
      if not (JArr.Items[i] is TJSONObject) then Continue;
      var JObj := TJSONObject(JArr.Items[i]);
      
      Result[i].ID := JObj.Get('id', 0);
      Result[i].Name := JObj.Get('name', '');
      Result[i].FullName := JObj.Get('full_name', '');
      Result[i].Description := JObj.Get('description', '');
      Result[i].URL := JObj.Get('html_url', '');
      Result[i].Stars := JObj.Get('stargazers_count', 0);
      Result[i].Forks := JObj.Get('forks_count', 0);
      Result[i].OpenIssues := JObj.Get('open_issues_count', 0);
      Result[i].Language := JObj.Get('language', '');
      Result[i].Private := JObj.Get('private', False);
      Result[i].CreatedAt := JObj.Get('created_at', '');
      Result[i].UpdatedAt := JObj.Get('updated_at', '');
      Result[i].DefaultBranch := JObj.Get('default_branch', '');
      Result[i].Size := JObj.Get('size', 0);
    end;
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Parse repos error: ' + E.Message;
  end;
end;

function TForm1.GetIssues(const Owner, Repo: string): TArray<TGitHubIssue>;
var
  Response: string;
  JArr: TJSONArray;
  i: Integer;
begin
  SetLength(Result, 0);
  
  Response := GetRequest(Format('/repos/%s/%s/issues?state=open&per_page=50', [Owner, Repo]));
  if Response = '' then Exit;
  
  try
    var JData := GetJSON(Response);
    if not (JData is TJSONArray) then Exit;
    JArr := TJSONArray(JData);
    
    SetLength(Result, JArr.Count);
    for i := 0 to JArr.Count - 1 do
    begin
      if not (JArr.Items[i] is TJSONObject) then Continue;
      var JObj := TJSONObject(JArr.Items[i]);
      
      Result[i].Number := JObj.Get('number', 0);
      Result[i].Title := JObj.Get('title', '');
      Result[i].State := JObj.Get('state', '');
      Result[i].Body := JObj.Get('body', '');
      Result[i].CreatedAt := JObj.Get('created_at', '');
      
      var JUser := JObj.Find('user');
      if JUser is TJSONObject then
        Result[i].Author := TJSONObject(JUser).Get('login', '');
    end;
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Parse issues error: ' + E.Message;
  end;
end;

function TForm1.CreateRepo(const Name, Description: string; Private_: Boolean): TGitHubRepo;
var
  Body, Response: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  Body := Format('{"name":"%s","description":"%s","private":%s,"auto_init":true}',
    [Name, Description, BoolToStr(Private_, 'true', 'false')]);
  
  Response := PostRequest('/user/repos', Body);
  if Response = '' then Exit;
  
  try
    var JData := GetJSON(Response);
    if not (JData is TJSONObject) then Exit;
    var JObj := TJSONObject(JData);
    
    Result.ID := JObj.Get('id', 0);
    Result.Name := JObj.Get('name', '');
    Result.FullName := JObj.Get('full_name', '');
    Result.URL := JObj.Get('html_url', '');
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Create repo error: ' + E.Message;
  end;
end;

function TForm1.CreateIssue(const Owner, Repo, Title, Body: string): TGitHubIssue;
var
  ReqBody, Response: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  ReqBody := Format('{"title":"%s","body":"%s"}', [
    Title.Replace('"', '\"'),
    Body.Replace('"', '\"')
  ]);
  
  Response := PostRequest(Format('/repos/%s/%s/issues', [Owner, Repo]), ReqBody);
  if Response = '' then Exit;
  
  try
    var JData := GetJSON(Response);
    if not (JData is TJSONObject) then Exit;
    var JObj := TJSONObject(JData);
    
    Result.Number := JObj.Get('number', 0);
    Result.Title := JObj.Get('title', '');
    Result.State := JObj.Get('state', '');
    Result.CreatedAt := JObj.Get('created_at', '');
  except
    on E: Exception do
      StatusBar1.SimpleText := 'Create issue error: ' + E.Message;
  end;
end;

procedure TForm1.DisplayUser(const User: TGitHubUser);
begin
  lblUserLogin.Caption := '@' + User.Login;
  lblUserName.Caption := User.Name;
  lblUserBio.Caption := User.Bio;
  lblUserLocation.Caption := User.Location;
  lblUserRepos.Caption := Format('Repos: %d', [User.PublicRepos]);
  lblUserFollowers.Caption := Format('Followers: %d | Following: %d',
    [User.Followers, User.Following]);
end;

procedure TForm1.DisplayRepos(const Repos: TArray<TGitHubRepo>);
var
  i: Integer;
  Item: TListItem;
begin
  ListViewRepos.Clear;
  for i := 0 to High(Repos) do
  begin
    Item := ListViewRepos.Items.Add;
    Item.Caption := Repos[i].Name;
    if Repos[i].Private then Item.Caption := '🔒 ' + Item.Caption;
    Item.SubItems.Add(Repos[i].Language);
    Item.SubItems.Add(IntToStr(Repos[i].Stars));
    Item.SubItems.Add(IntToStr(Repos[i].Forks));
    Item.SubItems.Add(IntToStr(Repos[i].OpenIssues));
    Item.SubItems.Add(Copy(Repos[i].UpdatedAt, 1, 10));
  end;
  
  StatusBar1.SimpleText := Format('พบ %d repositories', [Length(Repos)]);
end;

procedure TForm1.DisplayIssues(const Issues: TArray<TGitHubIssue>);
var
  i: Integer;
  Item: TListItem;
begin
  ListViewIssues.Clear;
  for i := 0 to High(Issues) do
  begin
    Item := ListViewIssues.Items.Add;
    Item.Caption := '#' + IntToStr(Issues[i].Number);
    Item.SubItems.Add(Issues[i].Title);
    Item.SubItems.Add(Issues[i].State);
    Item.SubItems.Add(Issues[i].Author);
    Item.SubItems.Add(Copy(Issues[i].CreatedAt, 1, 10));
  end;
  
  StatusBar1.SimpleText := Format('พบ %d issues', [Length(Issues)]);
end;

procedure TForm1.btnLoadUserClick(Sender: TObject);
var
  Username: string;
  User: TGitHubUser;
begin
  Username := Trim(EditUsername.Text);
  if Username = '' then begin ShowMessage('กรุณาใส่ Username'); Exit; end;
  
  StatusBar1.SimpleText := 'กำลังโหลดข้อมูลผู้ใช้...';
  Application.ProcessMessages;
  
  User := GetUserInfo(Username);
  if User.Login <> '' then
  begin
    DisplayUser(User);
    StatusBar1.SimpleText := 'โหลดข้อมูลผู้ใช้สำเร็จ';
  end;
end;

procedure TForm1.btnListReposClick(Sender: TObject);
var
  Username: string;
  Repos: TArray<TGitHubRepo>;
begin
  Username := Trim(EditUsername.Text);
  if Username = '' then begin ShowMessage('กรุณาใส่ Username'); Exit; end;
  
  StatusBar1.SimpleText := 'กำลังโหลด Repositories...';
  Application.ProcessMessages;
  
  Repos := GetRepos(Username);
  DisplayRepos(Repos);
end;

procedure TForm1.btnCreateRepoClick(Sender: TObject);
var
  Name, Description: string;
  Repo: TGitHubRepo;
begin
  if Trim(EditToken.Text) = '' then
  begin
    ShowMessage('ต้องการ GitHub Token สำหรับสร้าง Repository');
    Exit;
  end;
  
  Name := '';
  if not InputQuery('สร้าง Repository', 'ชื่อ Repository:', Name) then Exit;
  
  Description := '';
  InputQuery('สร้าง Repository', 'คำอธิบาย:', Description);
  
  StatusBar1.SimpleText := 'กำลังสร้าง Repository...';
  Application.ProcessMessages;
  
  Repo := CreateRepo(Name, Description, False);
  if Repo.ID > 0 then
  begin
    ShowMessage(Format('สร้าง Repository "%s" สำเร็จ!', [Repo.FullName]));
    btnListReposClick(nil);
  end else
    StatusBar1.SimpleText := 'สร้าง Repository ไม่สำเร็จ';
end;

procedure TForm1.btnListIssuesClick(Sender: TObject);
var
  Owner, Repo: string;
  Issues: TArray<TGitHubIssue>;
begin
  Owner := Trim(EditUsername.Text);
  Repo := Trim(EditRepo.Text);
  
  if Owner = '' then begin ShowMessage('กรุณาใส่ Owner'); Exit; end;
  if Repo = '' then begin ShowMessage('กรุณาใส่ Repository'); Exit; end;
  
  StatusBar1.SimpleText := 'กำลังโหลด Issues...';
  Application.ProcessMessages;
  
  Issues := GetIssues(Owner, Repo);
  DisplayIssues(Issues);
end;

procedure TForm1.btnCreateIssueClick(Sender: TObject);
var
  Owner, Repo, Title, Body: string;
  Issue: TGitHubIssue;
begin
  if Trim(EditToken.Text) = '' then
  begin
    ShowMessage('ต้องการ GitHub Token สำหรับสร้าง Issue');
    Exit;
  end;
  
  Owner := Trim(EditUsername.Text);
  Repo := Trim(EditRepo.Text);
  
  if (Owner = '') or (Repo = '') then
  begin
    ShowMessage('กรุณาใส่ Owner และ Repository');
    Exit;
  end;
  
  Title := '';
  if not InputQuery('สร้าง Issue', 'หัวข้อ:', Title) then Exit;
  
  Body := '';
  InputQuery('สร้าง Issue', 'รายละเอียด:', Body);
  
  Issue := CreateIssue(Owner, Repo, Title, Body);
  if Issue.Number > 0 then
  begin
    ShowMessage(Format('สร้าง Issue #%d สำเร็จ: %s', [Issue.Number, Issue.Title]));
    btnListIssuesClick(nil);
  end else
    StatusBar1.SimpleText := 'สร้าง Issue ไม่สำเร็จ';
end;

end.
```

---

## แบบฝึกหัดบทที่ 40

### ข้อ 1: Currency Converter
สร้างโปรแกรมแปลงอัตราแลกเปลี่ยนเงิน โดยดึงข้อมูลจาก Exchange Rate API

```pascal
// แนวทาง: ใช้ exchangerate-api.com หรือ fixer.io
// GET https://api.exchangerate-api.com/v4/latest/THB
// Response: {"base":"THB","rates":{"USD":0.028,"EUR":0.026,...}}

function GetExchangeRates(const BaseCurrency: string): TJSONObject;
var
  HTTP: TIdHTTP;
  Response: string;
begin
  HTTP := TIdHTTP.Create(nil);
  try
    Response := HTTP.Get('https://api.exchangerate-api.com/v4/latest/' + BaseCurrency);
    var JData := GetJSON(Response);
    if JData is TJSONObject then
      Result := TJSONObject(TJSONObject(JData).Find('rates'))
    else
      Result := nil;
  finally
    HTTP.Free;
  end;
end;
```

### ข้อ 2: News Reader
สร้างโปรแกรมอ่านข่าวจาก NewsAPI (newsapi.org)

### ข้อ 3: JSON Placeholder CRUD
สร้าง CRUD application ด้วย JSONPlaceholder API (jsonplaceholder.typicode.com)

```pascal
// GET    https://jsonplaceholder.typicode.com/posts     - รายการ posts
// GET    https://jsonplaceholder.typicode.com/posts/1   - post #1
// POST   https://jsonplaceholder.typicode.com/posts     - สร้าง post ใหม่
// PUT    https://jsonplaceholder.typicode.com/posts/1   - แทนที่ post
// PATCH  https://jsonplaceholder.typicode.com/posts/1   - แก้ไข post
// DELETE https://jsonplaceholder.typicode.com/posts/1   - ลบ post
```

### ข้อ 4: Image Downloader
สร้างโปรแกรม Download รูปภาพหลายไฟล์พร้อมกัน (Multi-thread download)

```pascal
// แนวทาง: ใช้ TThread สำหรับแต่ละไฟล์
type
  TDownloadThread = class(TThread)
  private
    FURL, FSavePath: string;
    FOnComplete: procedure(URL, Path: string; Success: Boolean) of object;
  protected
    procedure Execute; override;
  end;
```

### ข้อ 5: REST API Mock Server
สร้าง Simple HTTP Server ที่ตอบสนอง REST API requests (JSON)

### ข้อ 6: OAuth2 Login
สร้างระบบ Login ด้วย Google OAuth2 

```pascal
// ขั้นตอน:
// 1. สร้าง Authorization URL
// 2. เปิด Browser ให้ User Login
// 3. รับ Authorization Code
// 4. แลก Code เป็น Access Token
// 5. ใช้ Token เรียก API

function BuildOAuth2URL(ClientID, RedirectURI, Scope: string): string;
begin
  Result := Format(
    'https://accounts.google.com/o/oauth2/auth?' +
    'client_id=%s&redirect_uri=%s&scope=%s&response_type=code',
    [URLEncode(ClientID), URLEncode(RedirectURI), URLEncode(Scope)]
  );
end;
```

### ข้อ 7: Rate Limiting Handler
สร้าง HTTP Client ที่จัดการ Rate Limit (429 Too Many Requests) โดยอัตโนมัติ

```pascal
// แนวทาง: จับ Exception และรอตามเวลาที่ API บอก
procedure RateLimitedGet(const URL: string; MaxRetries: Integer);
var
  i: Integer;
  RetryAfter: Integer;
begin
  for i := 1 to MaxRetries do
  begin
    try
      Result := HTTP.Get(URL);
      Break;  // สำเร็จ
    except
      on E: EIdHTTPProtocolException do
      begin
        if E.ErrorCode = 429 then
        begin
          RetryAfter := StrToIntDef(HTTP.Response.CustomHeaders.Values['Retry-After'], 60);
          Sleep(RetryAfter * 1000);
        end else
          raise;
      end;
    end;
  end;
end;
```

### ข้อ 8: Webhook Receiver
สร้าง HTTP Server รับ Webhook จาก GitHub, Stripe หรือ Service อื่น

### ข้อ 9: HTTP Proxy Client
สร้างโปรแกรมที่ส่ง Request ผ่าน HTTP Proxy

```pascal
// ใช้ Indy Proxy settings
FHTTP.ProxyParams.ProxyServer := '127.0.0.1';
FHTTP.ProxyParams.ProxyPort := 8080;
FHTTP.ProxyParams.ProxyUsername := 'user';
FHTTP.ProxyParams.ProxyPassword := 'pass';
```

### ข้อ 10: API Testing Tool
สร้าง Postman-like tool สำหรับทดสอบ REST API

---

## สรุปบทที่ 40

ในบทนี้เราได้เรียนรู้:
1. **HTTP Protocol** - Methods, Status Codes, Headers
2. **TIdHTTP** - Indy HTTP Client พร้อม SSL
3. **Synapse** - HTTP Client แบบเบา
4. **GET, POST, PUT, DELETE** - HTTP Methods ทั้งหมด
5. **Request Headers** - Custom Headers, User-Agent
6. **Authentication** - Basic, Bearer, API Key, HMAC
7. **File Upload** - Multipart Form Data
8. **Cookie Management** - TIdCookieManager
9. **Weather App** - ตัวอย่างจริงกับ OpenWeatherMap API
10. **GitHub API Client** - ตัวอย่างจริงกับ GitHub REST API

การเรียนรู้ HTTP Client จะช่วยให้โปรแกรม Lazarus ของคุณสามารถเชื่อมต่อกับ Web Services และ APIs ได้หลากหลาย เปิดโอกาสสร้างแอปพลิเคชันที่ทรงพลังและมีประโยชน์มากขึ้น

---

## บทสรุปของคอร์ส

ตลอดหลักสูตรนี้เราได้เรียนรู้ Lazarus/Pascal จากพื้นฐานถึงขั้นสูง:
- **Part 36**: Canvas และ Drawing - การวาดกราฟิกและ Animation
- **Part 37**: Image Processing - การประมวลผลรูปภาพ
- **Part 38**: Multimedia - เสียง วิดีโอ และ TTS
- **Part 39**: Network Programming - TCP/IP, Sockets, Chat
- **Part 40**: HTTP Client - REST API, การเชื่อมต่อ Web Services

ด้วยความรู้เหล่านี้ คุณสามารถสร้างโปรแกรมที่ซับซ้อนได้ ทั้ง Desktop Application, Network Application และ API Integration ขอให้โชคดีในการพัฒนาโปรแกรม!
