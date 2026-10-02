# ตอนที่ 70: Cloud Services Integration ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการเชื่อมต่อกับ Cloud Services ต่างๆ เช่น AWS S3, Google Cloud Storage, Azure Blob Storage รวมถึง OAuth authentication

---

## 70.1 REST API Client พื้นฐาน

```pascal
unit rest_client;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, HTTPSend, SSL_OpenSSL, fpjson, jsonparser;

type
  THTTPMethod = (hmGET, hmPOST, hmPUT, hmDELETE, hmPATCH);

  TRESTResponse = record
    StatusCode: Integer;
    StatusText: string;
    Body: string;
    Headers: TStringList;
    Success: Boolean;
  end;

  TRESTClient = class
  private
    FBaseURL: string;
    FHeaders: TStringList;
    FTimeout: Integer;
    FLastResponse: TRESTResponse;
    
    function DoRequest(AMethod: THTTPMethod; const AURL, ABody: string): TRESTResponse;
    function MethodToString(AMethod: THTTPMethod): string;
    
  public
    constructor Create(const ABaseURL: string);
    destructor Destroy; override;
    
    procedure SetHeader(const AName, AValue: string);
    procedure SetBearerToken(const AToken: string);
    procedure ClearHeaders;
    
    function Get(const APath: string): TRESTResponse;
    function Post(const APath, ABody: string): TRESTResponse;
    function Put(const APath, ABody: string): TRESTResponse;
    function Delete(const APath: string): TRESTResponse;
    function Patch(const APath, ABody: string): TRESTResponse;
    
    function ParseJSON(const AJSON: string): TJSONData;
    
    property BaseURL: string read FBaseURL write FBaseURL;
    property Timeout: Integer read FTimeout write FTimeout;
    property LastResponse: TRESTResponse read FLastResponse;
  end;

implementation

constructor TRESTClient.Create(const ABaseURL: string);
begin
  inherited Create;
  FBaseURL := ABaseURL;
  FHeaders := TStringList.Create;
  FTimeout := 30000;
  
  // Default headers
  FHeaders.Values['Accept'] := 'application/json';
  FHeaders.Values['Content-Type'] := 'application/json';
end;

destructor TRESTClient.Destroy;
begin
  FHeaders.Free;
  inherited Destroy;
end;

procedure TRESTClient.SetHeader(const AName, AValue: string);
begin
  FHeaders.Values[AName] := AValue;
end;

procedure TRESTClient.SetBearerToken(const AToken: string);
begin
  SetHeader('Authorization', 'Bearer ' + AToken);
end;

procedure TRESTClient.ClearHeaders;
begin
  FHeaders.Clear;
end;

function TRESTClient.MethodToString(AMethod: THTTPMethod): string;
begin
  case AMethod of
    hmGET:    Result := 'GET';
    hmPOST:   Result := 'POST';
    hmPUT:    Result := 'PUT';
    hmDELETE: Result := 'DELETE';
    hmPATCH:  Result := 'PATCH';
  end;
end;

function TRESTClient.DoRequest(AMethod: THTTPMethod; const AURL, ABody: string): TRESTResponse;
var
  HTTP: THTTPSend;
  BodyStream: TStringStream;
  ResponseBody: string;
  i: Integer;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.Headers := TStringList.Create;
  
  HTTP := THTTPSend.Create;
  try
    // Copy headers
    for i := 0 to FHeaders.Count - 1 do
      HTTP.Headers.Add(FHeaders[i]);
    
    HTTP.Timeout := FTimeout;
    
    // Add body for POST/PUT/PATCH
    if (ABody <> '') and (AMethod in [hmPOST, hmPUT, hmPATCH]) then
    begin
      BodyStream := TStringStream.Create(ABody);
      try
        HTTP.Document.CopyFrom(BodyStream, BodyStream.Size);
      finally
        BodyStream.Free;
      end;
    end;
    
    // Execute request
    if HTTP.HTTPMethod(MethodToString(AMethod), AURL) then
    begin
      Result.StatusCode := HTTP.ResultCode;
      Result.StatusText := HTTP.ResultString;
      Result.Success := HTTP.ResultCode in [200, 201, 202, 204];
      
      // Read response body
      if HTTP.Document.Size > 0 then
      begin
        SetLength(ResponseBody, HTTP.Document.Size);
        HTTP.Document.Position := 0;
        HTTP.Document.Read(ResponseBody[1], HTTP.Document.Size);
        Result.Body := ResponseBody;
      end;
      
      // Copy response headers
      Result.Headers.Assign(HTTP.Headers);
    end
    else
    begin
      Result.StatusCode := 0;
      Result.StatusText := 'Connection failed';
      Result.Success := False;
    end;
    
  finally
    HTTP.Free;
  end;
  
  FLastResponse := Result;
end;

function TRESTClient.Get(const APath: string): TRESTResponse;
begin
  Result := DoRequest(hmGET, FBaseURL + APath, '');
end;

function TRESTClient.Post(const APath, ABody: string): TRESTResponse;
begin
  Result := DoRequest(hmPOST, FBaseURL + APath, ABody);
end;

function TRESTClient.Put(const APath, ABody: string): TRESTResponse;
begin
  Result := DoRequest(hmPUT, FBaseURL + APath, ABody);
end;

function TRESTClient.Delete(const APath: string): TRESTResponse;
begin
  Result := DoRequest(hmDELETE, FBaseURL + APath, '');
end;

function TRESTClient.Patch(const APath, ABody: string): TRESTResponse;
begin
  Result := DoRequest(hmPATCH, FBaseURL + APath, ABody);
end;

function TRESTClient.ParseJSON(const AJSON: string): TJSONData;
var
  Parser: TJSONParser;
begin
  Result := nil;
  if AJSON = '' then Exit;
  
  Parser := TJSONParser.Create(AJSON, []);
  try
    Result := Parser.Parse;
  finally
    Parser.Free;
  end;
end;

end.
```

---

## 70.2 AWS S3 Integration

```pascal
unit aws_s3;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils, HTTPSend, SSL_OpenSSL,
  sha2, hmac, base16;  // ต้องติดตั้ง library เพิ่ม

type
  TAWSS3Config = record
    AccessKeyID: string;
    SecretAccessKey: string;
    Region: string;
    BucketName: string;
    Endpoint: string;  // '' = default AWS endpoint
  end;

  TS3Object = record
    Key: string;
    Size: Int64;
    LastModified: TDateTime;
    ETag: string;
    ContentType: string;
  end;

  TAWSS3Client = class
  private
    FConfig: TAWSS3Config;
    
    function GetEndpoint: string;
    function GetBucketURL: string;
    function CreateSignature(const AMethod, APath, APayloadHash: string;
      ADate: TDateTime; var AHeaders: TStringList): string;
    function SHA256Hash(const AData: string): string;
    function HMACSign(const AKey, AData: string): string;
    function BuildAuthHeader(const AMethod, APath: string; 
      ADate: TDateTime; AHeaders: TStringList): string;
    
  public
    constructor Create(const AConfig: TAWSS3Config);
    
    function ListObjects(const APrefix: string = ''): TList;
    function UploadFile(const AKey, AFilePath: string): Boolean;
    function UploadData(const AKey: string; AData: TStream; 
      const AContentType: string = 'application/octet-stream'): Boolean;
    function DownloadFile(const AKey, ADestPath: string): Boolean;
    function DownloadData(const AKey: string; AStream: TStream): Boolean;
    function DeleteObject(const AKey: string): Boolean;
    function GetPresignedURL(const AKey: string; AExpirationSec: Integer = 3600): string;
    function ObjectExists(const AKey: string): Boolean;
    function GetObjectInfo(const AKey: string; out AInfo: TS3Object): Boolean;
    
    property Config: TAWSS3Config read FConfig;
  end;

// Helper functions
function DefaultS3Config(const AAccessKey, ASecretKey, ARegion, ABucket: string): TAWSS3Config;

implementation

function DefaultS3Config(const AAccessKey, ASecretKey, ARegion, ABucket: string): TAWSS3Config;
begin
  Result.AccessKeyID := AAccessKey;
  Result.SecretAccessKey := ASecretKey;
  Result.Region := ARegion;
  Result.BucketName := ABucket;
  Result.Endpoint := '';
end;

constructor TAWSS3Client.Create(const AConfig: TAWSS3Config);
begin
  inherited Create;
  FConfig := AConfig;
end;

function TAWSS3Client.GetEndpoint: string;
begin
  if FConfig.Endpoint <> '' then
    Result := FConfig.Endpoint
  else
    Result := Format('https://s3.%s.amazonaws.com', [FConfig.Region]);
end;

function TAWSS3Client.GetBucketURL: string;
begin
  if FConfig.Endpoint <> '' then
    Result := FConfig.Endpoint + '/' + FConfig.BucketName
  else
    Result := Format('https://%s.s3.%s.amazonaws.com', 
      [FConfig.BucketName, FConfig.Region]);
end;

function TAWSS3Client.SHA256Hash(const AData: string): string;
begin
  // ใช้ library SHA256 จริงๆ
  // ตัวอย่างนี้เป็น placeholder
  Result := 'e3b0c44298fc1c149afb...';  // SHA256 ของ empty string
end;

function TAWSS3Client.UploadFile(const AKey, AFilePath: string): Boolean;
var
  FileStream: TFileStream;
begin
  Result := False;
  
  if not FileExists(AFilePath) then
  begin
    WriteLn('ไม่พบไฟล์: ', AFilePath);
    Exit;
  end;
  
  FileStream := TFileStream.Create(AFilePath, fmOpenRead or fmShareDenyNone);
  try
    Result := UploadData(AKey, FileStream, 'application/octet-stream');
  finally
    FileStream.Free;
  end;
end;

function TAWSS3Client.UploadData(const AKey: string; AData: TStream; 
  const AContentType: string): Boolean;
var
  HTTP: THTTPSend;
  URL: string;
  UploadDate: TDateTime;
  DateStr: string;
begin
  Result := False;
  URL := GetBucketURL + '/' + AKey;
  UploadDate := Now;
  DateStr := FormatDateTime('yyyymmdd"T"hhnnss"Z"', UploadDate);
  
  HTTP := THTTPSend.Create;
  try
    // AWS Signature Version 4 headers
    HTTP.Headers.Add('x-amz-date: ' + DateStr);
    HTTP.Headers.Add('x-amz-content-sha256: ' + SHA256Hash(''));
    HTTP.Headers.Add('Content-Type: ' + AContentType);
    HTTP.Headers.Add('Host: ' + FConfig.BucketName + '.s3.' + 
      FConfig.Region + '.amazonaws.com');
    
    // TODO: Add proper AWS Signature V4
    // ในการใช้จริงต้องใช้ library หรือ implement signature ให้ครบ
    
    AData.Position := 0;
    HTTP.Document.CopyFrom(AData, AData.Size);
    
    if HTTP.HTTPMethod('PUT', URL) then
      Result := HTTP.ResultCode in [200, 201]
    else
      WriteLn('Upload failed: connection error');
      
  finally
    HTTP.Free;
  end;
end;

function TAWSS3Client.DownloadFile(const AKey, ADestPath: string): Boolean;
var
  FileStream: TFileStream;
begin
  Result := False;
  
  FileStream := TFileStream.Create(ADestPath, fmCreate);
  try
    Result := DownloadData(AKey, FileStream);
  finally
    FileStream.Free;
  end;
  
  if not Result then
    DeleteFile(ADestPath);
end;

function TAWSS3Client.DownloadData(const AKey: string; AStream: TStream): Boolean;
var
  HTTP: THTTPSend;
  URL: string;
begin
  Result := False;
  URL := GetBucketURL + '/' + AKey;
  
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('GET', URL) then
    begin
      if HTTP.ResultCode = 200 then
      begin
        AStream.CopyFrom(HTTP.Document, HTTP.Document.Size);
        Result := True;
      end
      else
        WriteLn('Download failed: ', HTTP.ResultCode);
    end;
  finally
    HTTP.Free;
  end;
end;

function TAWSS3Client.DeleteObject(const AKey: string): Boolean;
var
  HTTP: THTTPSend;
  URL: string;
begin
  Result := False;
  URL := GetBucketURL + '/' + AKey;
  
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('DELETE', URL) then
      Result := HTTP.ResultCode = 204;
  finally
    HTTP.Free;
  end;
end;

function TAWSS3Client.ObjectExists(const AKey: string): Boolean;
var
  HTTP: THTTPSend;
  URL: string;
begin
  Result := False;
  URL := GetBucketURL + '/' + AKey;
  
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('HEAD', URL) then
      Result := HTTP.ResultCode = 200;
  finally
    HTTP.Free;
  end;
end;

function TAWSS3Client.ListObjects(const APrefix: string): TList;
begin
  Result := TList.Create;
  WriteLn('ListObjects: ', FConfig.BucketName, '/', APrefix);
  // TODO: Parse XML response from S3 ListObjectsV2
end;

function TAWSS3Client.GetPresignedURL(const AKey: string; AExpirationSec: Integer): string;
begin
  // Pre-signed URL สำหรับ temporary access
  Result := Format('%s/%s?X-Amz-Expires=%d&X-Amz-Signature=...',
    [GetBucketURL, AKey, AExpirationSec]);
  WriteLn('Presigned URL created for: ', AKey);
end;

function TAWSS3Client.GetObjectInfo(const AKey: string; out AInfo: TS3Object): Boolean;
var
  HTTP: THTTPSend;
  URL: string;
  i: Integer;
begin
  Result := False;
  FillChar(AInfo, SizeOf(AInfo), 0);
  
  URL := GetBucketURL + '/' + AKey;
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('HEAD', URL) then
    begin
      if HTTP.ResultCode = 200 then
      begin
        AInfo.Key := AKey;
        
        for i := 0 to HTTP.Headers.Count - 1 do
        begin
          var Header := HTTP.Headers[i];
          if LowerCase(Copy(Header, 1, 15)) = 'content-length:' then
            AInfo.Size := StrToInt64Def(Trim(Copy(Header, 16, MaxInt)), 0);
          if LowerCase(Copy(Header, 1, 13)) = 'content-type:' then
            AInfo.ContentType := Trim(Copy(Header, 14, MaxInt));
          if LowerCase(Copy(Header, 1, 5)) = 'etag:' then
            AInfo.ETag := Trim(Copy(Header, 6, MaxInt));
        end;
        
        Result := True;
      end;
    end;
  finally
    HTTP.Free;
  end;
end;

end.
```

---

## 70.3 OAuth 2.0 Authentication

```pascal
unit oauth2_client;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, HTTPSend, SSL_OpenSSL, fpjson, jsonparser,
  base64, DateUtils;

type
  TOAuthConfig = record
    ClientID: string;
    ClientSecret: string;
    AuthorizationURL: string;
    TokenURL: string;
    RedirectURI: string;
    Scope: string;
  end;

  TOAuthToken = record
    AccessToken: string;
    RefreshToken: string;
    TokenType: string;
    ExpiresIn: Integer;
    ExpiresAt: TDateTime;
    Scope: string;
  end;

  TOAuthClient = class
  private
    FConfig: TOAuthConfig;
    FToken: TOAuthToken;
    
    function ExchangeCode(const ACode: string): Boolean;
    function DoTokenRequest(const AData: string): Boolean;
    function ParseTokenResponse(const AJSON: string): Boolean;
    
  public
    constructor Create(const AConfig: TOAuthConfig);
    
    function GetAuthorizationURL(const AState: string = ''): string;
    function HandleCallback(const ACode, AState: string): Boolean;
    function RefreshAccessToken: Boolean;
    function IsTokenValid: Boolean;
    
    procedure SetToken(const AToken: TOAuthToken);
    function GetValidToken: string;  // Auto-refresh if needed
    
    property Config: TOAuthConfig read FConfig;
    property Token: TOAuthToken read FToken;
  end;

// Provider configs
function GoogleOAuthConfig(const AClientID, AClientSecret, ARedirectURI: string): TOAuthConfig;
function GithubOAuthConfig(const AClientID, AClientSecret, ARedirectURI: string): TOAuthConfig;
function MicrosoftOAuthConfig(const AClientID, AClientSecret, ARedirectURI, ATenantID: string): TOAuthConfig;

implementation

function GoogleOAuthConfig(const AClientID, AClientSecret, ARedirectURI: string): TOAuthConfig;
begin
  Result.ClientID := AClientID;
  Result.ClientSecret := AClientSecret;
  Result.RedirectURI := ARedirectURI;
  Result.AuthorizationURL := 'https://accounts.google.com/o/oauth2/v2/auth';
  Result.TokenURL := 'https://oauth2.googleapis.com/token';
  Result.Scope := 'openid email profile';
end;

function GithubOAuthConfig(const AClientID, AClientSecret, ARedirectURI: string): TOAuthConfig;
begin
  Result.ClientID := AClientID;
  Result.ClientSecret := AClientSecret;
  Result.RedirectURI := ARedirectURI;
  Result.AuthorizationURL := 'https://github.com/login/oauth/authorize';
  Result.TokenURL := 'https://github.com/login/oauth/access_token';
  Result.Scope := 'read:user user:email';
end;

function MicrosoftOAuthConfig(const AClientID, AClientSecret, ARedirectURI, ATenantID: string): TOAuthConfig;
begin
  Result.ClientID := AClientID;
  Result.ClientSecret := AClientSecret;
  Result.RedirectURI := ARedirectURI;
  Result.AuthorizationURL := Format(
    'https://login.microsoftonline.com/%s/oauth2/v2.0/authorize', [ATenantID]);
  Result.TokenURL := Format(
    'https://login.microsoftonline.com/%s/oauth2/v2.0/token', [ATenantID]);
  Result.Scope := 'openid email profile offline_access';
end;

{ TOAuthClient }

constructor TOAuthClient.Create(const AConfig: TOAuthConfig);
begin
  inherited Create;
  FConfig := AConfig;
  FillChar(FToken, SizeOf(FToken), 0);
end;

function TOAuthClient.GetAuthorizationURL(const AState: string): string;
begin
  Result := FConfig.AuthorizationURL + '?' +
    'client_id=' + URLEncode(FConfig.ClientID) +
    '&redirect_uri=' + URLEncode(FConfig.RedirectURI) +
    '&response_type=code' +
    '&scope=' + URLEncode(FConfig.Scope);
    
  if AState <> '' then
    Result := Result + '&state=' + URLEncode(AState);
end;

function TOAuthClient.DoTokenRequest(const AData: string): Boolean;
var
  HTTP: THTTPSend;
  Body: TStringStream;
  Response: string;
begin
  Result := False;
  
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('Content-Type: application/x-www-form-urlencoded');
    HTTP.Headers.Add('Accept: application/json');
    
    Body := TStringStream.Create(AData);
    try
      HTTP.Document.CopyFrom(Body, Body.Size);
    finally
      Body.Free;
    end;
    
    if HTTP.HTTPMethod('POST', FConfig.TokenURL) then
    begin
      if HTTP.ResultCode in [200, 201] then
      begin
        SetLength(Response, HTTP.Document.Size);
        HTTP.Document.Position := 0;
        HTTP.Document.Read(Response[1], HTTP.Document.Size);
        
        Result := ParseTokenResponse(Response);
      end
      else
        WriteLn('Token request failed: ', HTTP.ResultCode);
    end;
  finally
    HTTP.Free;
  end;
end;

function TOAuthClient.ParseTokenResponse(const AJSON: string): Boolean;
var
  Parser: TJSONParser;
  Data: TJSONData;
  Obj: TJSONObject;
begin
  Result := False;
  
  Parser := TJSONParser.Create(AJSON, []);
  try
    Data := Parser.Parse;
    try
      if Data is TJSONObject then
      begin
        Obj := TJSONObject(Data);
        
        FToken.AccessToken := Obj.Get('access_token', '');
        FToken.RefreshToken := Obj.Get('refresh_token', '');
        FToken.TokenType := Obj.Get('token_type', 'Bearer');
        FToken.ExpiresIn := Obj.Get('expires_in', 3600);
        FToken.Scope := Obj.Get('scope', '');
        FToken.ExpiresAt := Now + FToken.ExpiresIn / 86400;
        
        Result := FToken.AccessToken <> '';
        
        if Result then
          WriteLn('OAuth token received, expires in ', FToken.ExpiresIn, ' seconds')
        else
          WriteLn('OAuth error: ', Obj.Get('error', ''), ' - ', 
            Obj.Get('error_description', ''));
      end;
    finally
      Data.Free;
    end;
  finally
    Parser.Free;
  end;
end;

function TOAuthClient.HandleCallback(const ACode, AState: string): Boolean;
begin
  Result := ExchangeCode(ACode);
end;

function TOAuthClient.ExchangeCode(const ACode: string): Boolean;
var
  PostData: string;
begin
  PostData := 'grant_type=authorization_code' +
              '&code=' + URLEncode(ACode) +
              '&redirect_uri=' + URLEncode(FConfig.RedirectURI) +
              '&client_id=' + URLEncode(FConfig.ClientID) +
              '&client_secret=' + URLEncode(FConfig.ClientSecret);
              
  Result := DoTokenRequest(PostData);
end;

function TOAuthClient.RefreshAccessToken: Boolean;
var
  PostData: string;
begin
  Result := False;
  
  if FToken.RefreshToken = '' then
  begin
    WriteLn('ไม่มี refresh token');
    Exit;
  end;
  
  PostData := 'grant_type=refresh_token' +
              '&refresh_token=' + URLEncode(FToken.RefreshToken) +
              '&client_id=' + URLEncode(FConfig.ClientID) +
              '&client_secret=' + URLEncode(FConfig.ClientSecret);
              
  Result := DoTokenRequest(PostData);
  
  if Result then
    WriteLn('Access token refreshed successfully')
  else
    WriteLn('Token refresh failed');
end;

function TOAuthClient.IsTokenValid: Boolean;
begin
  Result := (FToken.AccessToken <> '') and 
            (Now < FToken.ExpiresAt - (5 / (24 * 60)));  // 5 minutes buffer
end;

procedure TOAuthClient.SetToken(const AToken: TOAuthToken);
begin
  FToken := AToken;
end;

function TOAuthClient.GetValidToken: string;
begin
  if not IsTokenValid then
  begin
    if FToken.RefreshToken <> '' then
      RefreshAccessToken
    else
      WriteLn('Token expired and no refresh token available');
  end;
  
  Result := FToken.AccessToken;
end;

end.
```

---

## 70.4 File Backup System

```pascal
unit cloud_backup;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils,
  aws_s3, rest_client;

type
  TBackupConfig = record
    Provider: string;        // 'aws', 'gcs', 'azure'
    LocalPath: string;       // โฟลเดอร์ที่จะ backup
    RemotePath: string;      // prefix ใน cloud storage
    Encrypt: Boolean;        // เข้ารหัสก่อน upload
    CompressFiles: Boolean;  // บีบอัดก่อน upload
    RetentionDays: Integer;  // เก็บ backup กี่วัน
    IncludePattern: string;  // Pattern ของไฟล์ที่จะ backup
  end;

  TBackupStats = record
    TotalFiles: Integer;
    UploadedFiles: Integer;
    SkippedFiles: Integer;
    FailedFiles: Integer;
    TotalBytes: Int64;
    UploadedBytes: Int64;
    StartTime: TDateTime;
    EndTime: TDateTime;
  end;

  TBackupProgressEvent = procedure(const AFileName: string; 
    APercent: Integer) of object;

  TCloudBackup = class
  private
    FConfig: TBackupConfig;
    FS3: TAWSS3Client;
    FStats: TBackupStats;
    FOnProgress: TBackupProgressEvent;
    FLogFile: TextFile;
    FLogEnabled: Boolean;
    
    procedure LogMessage(const AMsg: string);
    function ShouldBackupFile(const AFileName: string): Boolean;
    function GetRemoteKey(const ALocalPath: string): string;
    function BackupFile(const AFilePath: string): Boolean;
    procedure ScanDirectory(const APath: string; AFiles: TStringList);
    procedure CleanOldBackups;
    
  public
    constructor Create(const AConfig: TBackupConfig; AS3: TAWSS3Client);
    destructor Destroy; override;
    
    procedure EnableLogging(const ALogPath: string);
    procedure DisableLogging;
    
    function RunBackup: TBackupStats;
    function RunIncrementalBackup: TBackupStats;
    
    procedure ScheduleBackup(AHour, AMinute: Integer);  // Simple scheduling
    
    property Config: TBackupConfig read FConfig;
    property Stats: TBackupStats read FStats;
    property OnProgress: TBackupProgressEvent read FOnProgress write FOnProgress;
  end;

implementation

constructor TCloudBackup.Create(const AConfig: TBackupConfig; AS3: TAWSS3Client);
begin
  inherited Create;
  FConfig := AConfig;
  FS3 := AS3;
  FLogEnabled := False;
end;

destructor TCloudBackup.Destroy;
begin
  DisableLogging;
  inherited Destroy;
end;

procedure TCloudBackup.EnableLogging(const ALogPath: string);
begin
  AssignFile(FLogFile, ALogPath);
  if FileExists(ALogPath) then
    Append(FLogFile)
  else
    Rewrite(FLogFile);
  FLogEnabled := True;
  LogMessage('=== Backup started at ' + DateTimeToStr(Now) + ' ===');
end;

procedure TCloudBackup.DisableLogging;
begin
  if FLogEnabled then
  begin
    LogMessage('=== Backup ended at ' + DateTimeToStr(Now) + ' ===');
    CloseFile(FLogFile);
    FLogEnabled := False;
  end;
end;

procedure TCloudBackup.LogMessage(const AMsg: string);
begin
  WriteLn(Format('[%s] %s', [FormatDateTime('hh:nn:ss', Now), AMsg]));
  
  if FLogEnabled then
    WriteLn(FLogFile, Format('[%s] %s', [FormatDateTime('yyyy-mm-dd hh:nn:ss', Now), AMsg]));
end;

function TCloudBackup.ShouldBackupFile(const AFileName: string): Boolean;
var
  Pattern: string;
begin
  Result := True;
  
  // ตรวจสอบ pattern
  if FConfig.IncludePattern <> '' then
  begin
    Pattern := FConfig.IncludePattern;
    
    // Simple pattern matching (*.txt, *.pas, etc.)
    if Copy(Pattern, 1, 2) = '*.' then
    begin
      var Ext := Copy(Pattern, 3, MaxInt);
      Result := LowerCase(ExtractFileExt(AFileName)) = '.' + LowerCase(Ext);
    end
    else
      Result := Pos(Pattern, AFileName) > 0;
  end;
  
  // ข้ามไฟล์ temporary
  if Result then
  begin
    var Ext := LowerCase(ExtractFileExt(AFileName));
    if (Ext = '.tmp') or (Ext = '.bak') or 
       (Pos('~', AFileName) > 0) then
      Result := False;
  end;
end;

function TCloudBackup.GetRemoteKey(const ALocalPath: string): string;
var
  RelPath: string;
begin
  // แปลง path ท้องถิ่นเป็น key ใน cloud
  RelPath := StringReplace(ALocalPath, FConfig.LocalPath, '', []);
  
  // แปลง \ เป็น /
  RelPath := StringReplace(RelPath, '\', '/', [rfReplaceAll]);
  
  // ลบ / นำหน้า
  if (RelPath <> '') and (RelPath[1] = '/') then
    Delete(RelPath, 1, 1);
    
  // เพิ่ม timestamp folder สำหรับ versioning
  Result := FConfig.RemotePath + '/' + 
            FormatDateTime('yyyy-mm-dd', Now) + '/' + RelPath;
end;

function TCloudBackup.BackupFile(const AFilePath: string): Boolean;
var
  RemoteKey: string;
  FileSize: Int64;
begin
  Result := False;
  
  if not FileExists(AFilePath) then
  begin
    LogMessage('File not found: ' + AFilePath);
    Exit;
  end;
  
  FileSize := FileSize(AFilePath);
  RemoteKey := GetRemoteKey(AFilePath);
  
  LogMessage(Format('Uploading: %s -> %s (%d bytes)', 
    [ExtractFileName(AFilePath), RemoteKey, FileSize]));
    
  if Assigned(FOnProgress) then
    FOnProgress(ExtractFileName(AFilePath), 0);
  
  Result := FS3.UploadFile(RemoteKey, AFilePath);
  
  if Result then
  begin
    Inc(FStats.UploadedFiles);
    Inc(FStats.UploadedBytes, FileSize);
    LogMessage('Upload success: ' + ExtractFileName(AFilePath));
    
    if Assigned(FOnProgress) then
      FOnProgress(ExtractFileName(AFilePath), 100);
  end
  else
  begin
    Inc(FStats.FailedFiles);
    LogMessage('Upload FAILED: ' + AFilePath);
  end;
end;

procedure TCloudBackup.ScanDirectory(const APath: string; AFiles: TStringList);
var
  SR: TSearchRec;
begin
  if FindFirst(APath + DirectorySeparator + '*', faAnyFile, SR) = 0 then
  try
    repeat
      if (SR.Name <> '.') and (SR.Name <> '..') then
      begin
        var FullPath := APath + DirectorySeparator + SR.Name;
        
        if (SR.Attr and faDirectory) <> 0 then
          ScanDirectory(FullPath, AFiles)
        else if ShouldBackupFile(SR.Name) then
          AFiles.Add(FullPath);
      end;
    until FindNext(SR) <> 0;
  finally
    FindClose(SR);
  end;
end;

function TCloudBackup.RunBackup: TBackupStats;
var
  Files: TStringList;
  i: Integer;
begin
  FillChar(FStats, SizeOf(FStats), 0);
  FStats.StartTime := Now;
  
  LogMessage('Starting full backup of: ' + FConfig.LocalPath);
  
  if not DirectoryExists(FConfig.LocalPath) then
  begin
    LogMessage('Source directory not found: ' + FConfig.LocalPath);
    FStats.EndTime := Now;
    Result := FStats;
    Exit;
  end;
  
  Files := TStringList.Create;
  try
    ScanDirectory(FConfig.LocalPath, Files);
    FStats.TotalFiles := Files.Count;
    
    // Calculate total size
    for i := 0 to Files.Count - 1 do
      Inc(FStats.TotalBytes, FileSize(Files[i]));
    
    LogMessage(Format('Found %d files (%d bytes)', 
      [FStats.TotalFiles, FStats.TotalBytes]));
    
    // Upload files
    for i := 0 to Files.Count - 1 do
    begin
      BackupFile(Files[i]);
      
      // Progress
      if Assigned(FOnProgress) then
        FOnProgress('Overall', Round(i * 100 / Files.Count));
    end;
    
  finally
    Files.Free;
  end;
  
  FStats.EndTime := Now;
  
  LogMessage(Format('Backup complete: %d/%d files uploaded in %.1f seconds',
    [FStats.UploadedFiles, FStats.TotalFiles,
     SecondsBetween(FStats.EndTime, FStats.StartTime)]));
     
  CleanOldBackups;
  
  Result := FStats;
end;

procedure TCloudBackup.CleanOldBackups;
begin
  if FConfig.RetentionDays <= 0 then Exit;
  
  LogMessage(Format('Cleaning backups older than %d days', [FConfig.RetentionDays]));
  // TODO: List objects in cloud and delete old ones
end;

function TCloudBackup.RunIncrementalBackup: TBackupStats;
begin
  // Incremental: backup only changed files
  LogMessage('Running incremental backup...');
  // TODO: Compare file timestamps with last backup manifest
  Result := RunBackup;  // Simplified: full backup
end;

end.
```

---

## 70.5 ตัวอย่างใช้งานจริง

```pascal
program cloud_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  aws_s3, cloud_backup, oauth2_client;

procedure DemoS3Upload;
var
  Config: TAWSS3Config;
  S3: TAWSS3Client;
  BackupConfig: TBackupConfig;
  Backup: TCloudBackup;
  Stats: TBackupStats;
begin
  WriteLn('=== AWS S3 Backup Demo ===');
  
  // ตั้งค่า AWS
  Config := DefaultS3Config(
    'YOUR_ACCESS_KEY',
    'YOUR_SECRET_KEY',
    'ap-southeast-1',  // Singapore
    'my-backup-bucket'
  );
  
  S3 := TAWSS3Client.Create(Config);
  try
    // ตั้งค่า backup
    BackupConfig.Provider := 'aws';
    BackupConfig.LocalPath := '/home/user/documents';
    BackupConfig.RemotePath := 'daily-backup';
    BackupConfig.Encrypt := False;
    BackupConfig.CompressFiles := False;
    BackupConfig.RetentionDays := 30;
    BackupConfig.IncludePattern := '*.pas';
    
    Backup := TCloudBackup.Create(BackupConfig, S3);
    try
      Backup.EnableLogging('/var/log/backup.log');
      
      Stats := Backup.RunBackup;
      
      WriteLn('');
      WriteLn('=== Backup Results ===');
      WriteLn(Format('Files: %d/%d uploaded', 
        [Stats.UploadedFiles, Stats.TotalFiles]));
      WriteLn(Format('Failed: %d files', [Stats.FailedFiles]));
      WriteLn(Format('Data: %.2f MB', 
        [Stats.UploadedBytes / (1024 * 1024)]));
      WriteLn(Format('Time: %.1f seconds',
        [SecondsBetween(Stats.EndTime, Stats.StartTime)]));
        
    finally
      Backup.Free;
    end;
    
  finally
    S3.Free;
  end;
end;

procedure DemoOAuth;
var
  Config: TOAuthConfig;
  Client: TOAuthClient;
  AuthURL: string;
  Code: string;
begin
  WriteLn('=== OAuth2 Demo ===');
  
  // Google OAuth
  Config := GoogleOAuthConfig(
    'your-client-id.apps.googleusercontent.com',
    'your-client-secret',
    'http://localhost:8080/callback'
  );
  
  Client := TOAuthClient.Create(Config);
  try
    AuthURL := Client.GetAuthorizationURL('random-state-123');
    WriteLn('เปิด URL นี้ในเบราว์เซอร์:');
    WriteLn(AuthURL);
    WriteLn('');
    
    Write('ใส่ authorization code จาก callback URL: ');
    ReadLn(Code);
    
    if Client.HandleCallback(Code, '') then
    begin
      WriteLn('OAuth สำเร็จ!');
      WriteLn('Access Token: ', Copy(Client.Token.AccessToken, 1, 20), '...');
    end
    else
      WriteLn('OAuth ล้มเหลว');
      
  finally
    Client.Free;
  end;
end;

begin
  // Demo S3 Upload
  DemoS3Upload;
  
  WriteLn('');
  
  // Demo OAuth
  DemoOAuth;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **REST API Client** - สร้าง HTTP client สำหรับ API calls
2. **AWS S3** - Upload/Download ไฟล์บน Amazon S3
3. **OAuth 2.0** - Authentication flow สำหรับ Google, GitHub, Microsoft
4. **Cloud Backup** - ระบบ backup ไฟล์อัตโนมัติ
5. **Cloud Integration Patterns** - Best practices สำหรับ cloud

Cloud services ช่วยให้แอปพลิเคชันมีความสามารถด้านการจัดเก็บข้อมูล, authentication, และ scalability ที่ดีกว่าการทำทุกอย่างเอง
