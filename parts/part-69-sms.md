# ตอนที่ 69: SMS Integration ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการส่ง SMS ผ่าน API ของ Gateway providers เช่น Twilio, การใช้ Modem ท้องถิ่น และการสร้างระบบ OTP

---

## 69.1 SMS Gateway API (Twilio)

```pascal
unit sms_twilio;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, HTTPSend, SSL_OpenSSL, base64;

type
  TSMSResult = record
    Success: Boolean;
    MessageSID: string;
    Status: string;
    ErrorCode: string;
    ErrorMessage: string;
  end;

  TTwilioSMS = class
  private
    FAccountSID: string;
    FAuthToken: string;
    FFromNumber: string;
    FBaseURL: string;
    
    function GetBasicAuth: string;
    function HTTPPost(const AURL, AData: string): string;
    function ParseResponse(const AJSON: string): TSMSResult;
    
  public
    constructor Create(const AAccountSID, AAuthToken, AFromNumber: string);
    
    function SendSMS(const AToNumber, AMessage: string): TSMSResult;
    function SendBulkSMS(const ANumbers: TStringList; const AMessage: string): Integer;
    function GetMessageStatus(const AMessageSID: string): string;
    
    property AccountSID: string read FAccountSID;
    property FromNumber: string read FFromNumber write FFromNumber;
  end;

implementation

uses
  fpjson, jsonparser;

constructor TTwilioSMS.Create(const AAccountSID, AAuthToken, AFromNumber: string);
begin
  inherited Create;
  FAccountSID := AAccountSID;
  FAuthToken := AAuthToken;
  FFromNumber := AFromNumber;
  FBaseURL := 'https://api.twilio.com/2010-04-01';
end;

function TTwilioSMS.GetBasicAuth: string;
var
  Credentials: string;
begin
  Credentials := FAccountSID + ':' + FAuthToken;
  Result := 'Basic ' + EncodeStringBase64(Credentials);
end;

function TTwilioSMS.HTTPPost(const AURL, AData: string): string;
var
  HTTP: THTTPSend;
  DataStream: TStringStream;
begin
  Result := '';
  HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('Authorization: ' + GetBasicAuth);
    HTTP.Headers.Add('Content-Type: application/x-www-form-urlencoded');
    HTTP.Headers.Add('Accept: application/json');
    
    DataStream := TStringStream.Create(AData);
    try
      HTTP.Document.CopyFrom(DataStream, DataStream.Size);
    finally
      DataStream.Free;
    end;
    
    if HTTP.HTTPMethod('POST', AURL) then
    begin
      if HTTP.ResultCode in [200, 201] then
      begin
        SetLength(Result, HTTP.Document.Size);
        HTTP.Document.Position := 0;
        HTTP.Document.Read(Result[1], HTTP.Document.Size);
      end
      else
      begin
        SetLength(Result, HTTP.Document.Size);
        HTTP.Document.Position := 0;
        HTTP.Document.Read(Result[1], HTTP.Document.Size);
        WriteLn('HTTP Error: ', HTTP.ResultCode, ' - ', Result);
      end;
    end
    else
      WriteLn('HTTP Error: ไม่สามารถเชื่อมต่อ');
  finally
    HTTP.Free;
  end;
end;

function TTwilioSMS.ParseResponse(const AJSON: string): TSMSResult;
var
  Parser: TJSONParser;
  Data: TJSONData;
  Obj: TJSONObject;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  if AJSON = '' then
  begin
    Result.ErrorMessage := 'ไม่ได้รับ response';
    Exit;
  end;
  
  Parser := TJSONParser.Create(AJSON, []);
  try
    Data := Parser.Parse;
    try
      if Data is TJSONObject then
      begin
        Obj := TJSONObject(Data);
        
        Result.MessageSID := Obj.Get('sid', '');
        Result.Status := Obj.Get('status', '');
        
        if Obj.Find('error_code') <> nil then
        begin
          Result.ErrorCode := Obj.Get('error_code', '');
          Result.ErrorMessage := Obj.Get('message', '');
          Result.Success := False;
        end
        else
          Result.Success := Result.MessageSID <> '';
      end;
    finally
      Data.Free;
    end;
  finally
    Parser.Free;
  end;
end;

function TTwilioSMS.SendSMS(const AToNumber, AMessage: string): TSMSResult;
var
  URL, PostData, Response: string;
begin
  URL := FBaseURL + '/Accounts/' + FAccountSID + '/Messages.json';
  
  PostData := 'From=' + URLEncode(FFromNumber) + 
              '&To=' + URLEncode(AToNumber) +
              '&Body=' + URLEncode(AMessage);
              
  Response := HTTPPost(URL, PostData);
  Result := ParseResponse(Response);
  
  if Result.Success then
    WriteLn('SMS ส่งสำเร็จ SID: ', Result.MessageSID)
  else
    WriteLn('SMS ส่งล้มเหลว: ', Result.ErrorMessage);
end;

function TTwilioSMS.SendBulkSMS(const ANumbers: TStringList; 
  const AMessage: string): Integer;
var
  i: Integer;
  Res: TSMSResult;
begin
  Result := 0;
  
  for i := 0 to ANumbers.Count - 1 do
  begin
    Res := SendSMS(ANumbers[i], AMessage);
    if Res.Success then Inc(Result);
    Sleep(100);  // Rate limiting
  end;
end;

function TTwilioSMS.GetMessageStatus(const AMessageSID: string): string;
var
  URL, Response: string;
  Parser: TJSONParser;
  Data: TJSONData;
begin
  Result := 'unknown';
  URL := FBaseURL + '/Accounts/' + FAccountSID + '/Messages/' + AMessageSID + '.json';
  
  // GET request
  var HTTP := THTTPSend.Create;
  try
    HTTP.Headers.Add('Authorization: ' + GetBasicAuth);
    
    if HTTP.HTTPMethod('GET', URL) then
    begin
      SetLength(Response, HTTP.Document.Size);
      HTTP.Document.Position := 0;
      HTTP.Document.Read(Response[1], HTTP.Document.Size);
      
      Parser := TJSONParser.Create(Response, []);
      try
        Data := Parser.Parse;
        try
          if Data is TJSONObject then
            Result := TJSONObject(Data).Get('status', 'unknown');
        finally
          Data.Free;
        end;
      finally
        Parser.Free;
      end;
    end;
  finally
    HTTP.Free;
  end;
end;

function URLEncode(const AStr: string): string;
var
  i: Integer;
  C: Char;
begin
  Result := '';
  for i := 1 to Length(AStr) do
  begin
    C := AStr[i];
    case C of
      'A'..'Z', 'a'..'z', '0'..'9', '-', '_', '.', '~':
        Result := Result + C;
      ' ':
        Result := Result + '+';
      else
        Result := Result + '%' + IntToHex(Ord(C), 2);
    end;
  end;
end;

end.
```

---

## 69.2 SMS Gateway ท้องถิ่น (Thailand)

```pascal
unit sms_thai_gateway;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, HTTPSend, SSL_OpenSSL;

// ตัวอย่าง: True Move SMS API
type
  TTrueMoveSMS = class
  private
    FAPIKey: string;
    FAPIURL: string;
    
    function SendRequest(const AParams: string): string;
    
  public
    constructor Create(const AAPIKey: string);
    
    function SendSMS(const APhone, AMessage: string): Boolean;
    function SendOTP(const APhone: string; out AOTP: string): Boolean;
    function VerifyOTP(const APhone, AOTP: string): Boolean;
  end;

// DTAC SMS API
type
  TDTACSendSMS = class
  private
    FUsername: string;
    FPassword: string;
    FAPIURL: string;
    
  public
    constructor Create(const AUsername, APassword: string);
    function SendSMS(const APhone, AMessage, ASenderID: string): Boolean;
  end;

// Generic Thai SMS Gateway
type
  TThaiSMSGateway = class
  private
    FProvider: string;
    FAPIKey: string;
    FSenderID: string;
    
    function BuildURL(const APhone, AMessage: string): string;
    
  public
    constructor Create(const AProvider, AAPIKey, ASenderID: string);
    
    function Send(const APhone, AMessage: string): Boolean;
    function SendBulk(const APhones: TStringList; const AMessage: string): Integer;
    
    // Format เบอร์โทรศัพท์ไทย
    class function FormatThaiPhone(const APhone: string): string;
    class function IsValidThaiPhone(const APhone: string): Boolean;
  end;

implementation

{ TThaiSMSGateway }

constructor TThaiSMSGateway.Create(const AProvider, AAPIKey, ASenderID: string);
begin
  inherited Create;
  FProvider := LowerCase(AProvider);
  FAPIKey := AAPIKey;
  FSenderID := ASenderID;
end;

class function TThaiSMSGateway.FormatThaiPhone(const APhone: string): string;
var
  Digits: string;
  i: Integer;
begin
  // เก็บแค่ตัวเลข
  Digits := '';
  for i := 1 to Length(APhone) do
    if APhone[i] in ['0'..'9'] then
      Digits := Digits + APhone[i];
  
  // แปลง 0XX -> 66XX
  if (Length(Digits) = 10) and (Digits[1] = '0') then
    Result := '66' + Copy(Digits, 2, 9)
  else if (Length(Digits) = 9) then
    Result := '66' + Digits
  else if (Length(Digits) = 11) and Copy(Digits, 1, 2) = '66' then
    Result := Digits
  else
    Result := Digits;
end;

class function TThaiSMSGateway.IsValidThaiPhone(const APhone: string): Boolean;
var
  Formatted: string;
begin
  Formatted := FormatThaiPhone(APhone);
  // เบอร์ไทย: 66XXXXXXXXX (12 หลัก)
  Result := (Length(Formatted) = 11) and 
            (Copy(Formatted, 1, 2) = '66');
end;

function TThaiSMSGateway.BuildURL(const APhone, AMessage: string): string;
begin
  case FProvider of
    'thsms':
      Result := Format('https://thsms.com/api/sms?key=%s&to=%s&from=%s&message=%s',
        [FAPIKey, URLEncode(APhone), URLEncode(FSenderID), URLEncode(AMessage)]);
        
    'buddymsg':
      Result := Format('https://www.buddymsg.com/api/?api_key=%s&msisdn=%s&message=%s&sender=%s',
        [FAPIKey, APhone, URLEncode(AMessage), FSenderID]);
        
    else  // Default generic
      Result := Format('https://sms.example.com/api?key=%s&to=%s&msg=%s',
        [FAPIKey, APhone, URLEncode(AMessage)]);
  end;
end;

function TThaiSMSGateway.Send(const APhone, AMessage: string): Boolean;
var
  FormattedPhone: string;
  URL, Response: string;
  HTTP: THTTPSend;
begin
  Result := False;
  
  FormattedPhone := FormatThaiPhone(APhone);
  
  if not IsValidThaiPhone(APhone) then
  begin
    WriteLn('เบอร์โทรไม่ถูกต้อง: ', APhone);
    Exit;
  end;
  
  URL := BuildURL(FormattedPhone, AMessage);
  
  HTTP := THTTPSend.Create;
  try
    if HTTP.HTTPMethod('GET', URL) then
    begin
      if HTTP.ResultCode = 200 then
      begin
        SetLength(Response, HTTP.Document.Size);
        HTTP.Document.Position := 0;
        HTTP.Document.Read(Response[1], HTTP.Document.Size);
        
        // ตรวจสอบ response
        Result := (Pos('success', LowerCase(Response)) > 0) or
                  (Pos('"status":1', LowerCase(Response)) > 0) or
                  (HTTP.ResultCode = 200);
                  
        WriteLn('SMS Response: ', Response);
      end;
    end;
  finally
    HTTP.Free;
  end;
end;

function TThaiSMSGateway.SendBulk(const APhones: TStringList; 
  const AMessage: string): Integer;
var
  i: Integer;
begin
  Result := 0;
  for i := 0 to APhones.Count - 1 do
  begin
    if Send(APhones[i], AMessage) then
      Inc(Result);
    Sleep(200);  // Rate limiting
  end;
  WriteLn(Format('ส่ง SMS %d/%d สำเร็จ', [Result, APhones.Count]));
end;

end.
```

---

## 69.3 OTP System

```pascal
unit otp_system;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils,
  sms_twilio;

type
  TOTPRecord = record
    Phone: string;
    OTP: string;
    CreatedAt: TDateTime;
    ExpiresAt: TDateTime;
    Attempts: Integer;
    IsVerified: Boolean;
    Purpose: string;  // 'login', 'register', 'payment', etc.
  end;

  TOTPConfig = record
    Length: Integer;          // จำนวนหลัก OTP
    ExpirationMinutes: Integer; // หมดอายุกี่นาที
    MaxAttempts: Integer;     // ลองผิดได้กี่ครั้ง
    AllowAlphaNumeric: Boolean; // ใช้ตัวอักษรด้วยหรือไม่
    MessageTemplate: string;  // Template ข้อความ
  end;

  TOTPManager = class
  private
    FRecords: TList;
    FSMS: TTwilioSMS;
    FConfig: TOTPConfig;
    
    function GenerateOTP: string;
    function FindRecord(const APhone: string): Integer;
    procedure CleanExpired;
    function BuildMessage(const AOTP: string): string;
    
  public
    constructor Create(ASMS: TTwilioSMS; AConfig: TOTPConfig);
    destructor Destroy; override;
    
    // ส่ง OTP
    function SendOTP(const APhone: string; const APurpose: string = 'login'): Boolean;
    function ResendOTP(const APhone: string): Boolean;
    
    // ยืนยัน OTP
    function VerifyOTP(const APhone, AOTP: string): Boolean;
    function GetOTPStatus(const APhone: string): string;
    
    // ยกเลิก OTP
    procedure InvalidateOTP(const APhone: string);
    
    // สถิติ
    function GetActiveCount: Integer;
    function GetStats: string;
    
    property Config: TOTPConfig read FConfig write FConfig;
  end;

  // Default configs
  function DefaultOTPConfig: TOTPConfig;
  function BankOTPConfig: TOTPConfig;

implementation

uses
  Math;

function DefaultOTPConfig: TOTPConfig;
begin
  Result.Length := 6;
  Result.ExpirationMinutes := 5;
  Result.MaxAttempts := 3;
  Result.AllowAlphaNumeric := False;
  Result.MessageTemplate := 'รหัส OTP ของคุณคือ {OTP} หมดอายุใน {EXPIRE} นาที ห้ามบอกใครเด็ดขาด';
end;

function BankOTPConfig: TOTPConfig;
begin
  Result.Length := 6;
  Result.ExpirationMinutes := 2;  // สั้นกว่า
  Result.MaxAttempts := 3;
  Result.AllowAlphaNumeric := False;
  Result.MessageTemplate := '[BANK] OTP: {OTP} ใช้ได้ {EXPIRE} นาที สำหรับการทำรายการ';
end;

{ TOTPManager }

constructor TOTPManager.Create(ASMS: TTwilioSMS; AConfig: TOTPConfig);
begin
  inherited Create;
  FRecords := TList.Create;
  FSMS := ASMS;
  FConfig := AConfig;
end;

destructor TOTPManager.Destroy;
var
  i: Integer;
begin
  for i := 0 to FRecords.Count - 1 do
    Dispose(POTPRecord(FRecords[i]));
  FRecords.Free;
  inherited Destroy;
end;

function TOTPManager.GenerateOTP: string;
const
  DIGITS = '0123456789';
  ALPHANUMERIC = '0123456789ABCDEFGHJKLMNPQRSTUVWXYZ';
var
  i: Integer;
  CharSet: string;
begin
  if FConfig.AllowAlphaNumeric then
    CharSet := ALPHANUMERIC
  else
    CharSet := DIGITS;
    
  Result := '';
  Randomize;
  
  for i := 1 to FConfig.Length do
    Result := Result + CharSet[1 + Random(Length(CharSet))];
end;

function TOTPManager.FindRecord(const APhone: string): Integer;
var
  i: Integer;
begin
  Result := -1;
  for i := 0 to FRecords.Count - 1 do
    if POTPRecord(FRecords[i])^.Phone = APhone then
    begin
      Result := i;
      Exit;
    end;
end;

procedure TOTPManager.CleanExpired;
var
  i: Integer;
  Rec: ^TOTPRecord;
begin
  for i := FRecords.Count - 1 downto 0 do
  begin
    Rec := FRecords[i];
    if (Now > Rec^.ExpiresAt) or Rec^.IsVerified then
    begin
      Dispose(Rec);
      FRecords.Delete(i);
    end;
  end;
end;

function TOTPManager.BuildMessage(const AOTP: string): string;
begin
  Result := FConfig.MessageTemplate;
  Result := StringReplace(Result, '{OTP}', AOTP, [rfReplaceAll]);
  Result := StringReplace(Result, '{EXPIRE}', 
    IntToStr(FConfig.ExpirationMinutes), [rfReplaceAll]);
end;

function TOTPManager.SendOTP(const APhone: string; const APurpose: string): Boolean;
var
  Idx: Integer;
  Rec: ^TOTPRecord;
  OTP: string;
  SMSResult: TSMSResult;
begin
  Result := False;
  CleanExpired;
  
  // ตรวจสอบว่ามี OTP ที่ยังใช้ได้อยู่หรือไม่
  Idx := FindRecord(APhone);
  if Idx >= 0 then
  begin
    Rec := FRecords[Idx];
    if (Now < Rec^.ExpiresAt) and not Rec^.IsVerified then
    begin
      // ส่ง OTP เดิม (ยังไม่หมดอายุ)
      SMSResult := FSMS.SendSMS(APhone, BuildMessage(Rec^.OTP));
      Result := SMSResult.Success;
      WriteLn('ส่ง OTP เดิมอีกครั้ง: ', APhone);
      Exit;
    end
    else
    begin
      // ลบ record เดิม
      Dispose(Rec);
      FRecords.Delete(Idx);
    end;
  end;
  
  // สร้าง OTP ใหม่
  OTP := GenerateOTP;
  
  // สร้าง record
  New(Rec);
  Rec^.Phone := APhone;
  Rec^.OTP := OTP;
  Rec^.CreatedAt := Now;
  Rec^.ExpiresAt := Now + FConfig.ExpirationMinutes / (24 * 60);
  Rec^.Attempts := 0;
  Rec^.IsVerified := False;
  Rec^.Purpose := APurpose;
  FRecords.Add(Rec);
  
  // ส่ง SMS
  SMSResult := FSMS.SendSMS(APhone, BuildMessage(OTP));
  Result := SMSResult.Success;
  
  if Result then
    WriteLn('ส่ง OTP สำเร็จ: ', APhone, ' (', APurpose, ')')
  else
  begin
    // ลบ record ถ้าส่งไม่สำเร็จ
    Dispose(Rec);
    FRecords.Delete(FRecords.Count - 1);
    WriteLn('ส่ง OTP ล้มเหลว: ', SMSResult.ErrorMessage);
  end;
end;

function TOTPManager.ResendOTP(const APhone: string): Boolean;
var
  Idx: Integer;
  Rec: ^TOTPRecord;
  SMSResult: TSMSResult;
begin
  Result := False;
  
  Idx := FindRecord(APhone);
  if Idx < 0 then
  begin
    WriteLn('ไม่พบ OTP สำหรับ: ', APhone);
    Exit;
  end;
  
  Rec := FRecords[Idx];
  
  if Now > Rec^.ExpiresAt then
  begin
    WriteLn('OTP หมดอายุแล้ว กรุณาขอใหม่');
    Exit;
  end;
  
  // ส่ง OTP เดิมอีกครั้ง
  SMSResult := FSMS.SendSMS(APhone, BuildMessage(Rec^.OTP));
  Result := SMSResult.Success;
  
  WriteLn('Resend OTP: ', APhone, ' - ', IfThen(Result, 'สำเร็จ', 'ล้มเหลว'));
end;

function TOTPManager.VerifyOTP(const APhone, AOTP: string): Boolean;
var
  Idx: Integer;
  Rec: ^TOTPRecord;
begin
  Result := False;
  CleanExpired;
  
  Idx := FindRecord(APhone);
  if Idx < 0 then
  begin
    WriteLn('ไม่พบ OTP สำหรับ: ', APhone);
    Exit;
  end;
  
  Rec := FRecords[Idx];
  
  // ตรวจสอบหมดอายุ
  if Now > Rec^.ExpiresAt then
  begin
    WriteLn('OTP หมดอายุแล้ว');
    Dispose(Rec);
    FRecords.Delete(Idx);
    Exit;
  end;
  
  // ตรวจสอบจำนวนครั้งที่ลอง
  if Rec^.Attempts >= FConfig.MaxAttempts then
  begin
    WriteLn('OTP ถูกล็อค (ลองผิดเกิน ', FConfig.MaxAttempts, ' ครั้ง)');
    Dispose(Rec);
    FRecords.Delete(Idx);
    Exit;
  end;
  
  Inc(Rec^.Attempts);
  
  // ตรวจสอบ OTP
  if Rec^.OTP = AOTP then
  begin
    Rec^.IsVerified := True;
    Result := True;
    WriteLn('OTP ถูกต้อง: ', APhone);
  end
  else
  begin
    WriteLn(Format('OTP ไม่ถูกต้อง (ครั้งที่ %d/%d)', 
      [Rec^.Attempts, FConfig.MaxAttempts]));
  end;
end;

function TOTPManager.GetOTPStatus(const APhone: string): string;
var
  Idx: Integer;
  Rec: ^TOTPRecord;
  MinLeft: Integer;
begin
  CleanExpired;
  Idx := FindRecord(APhone);
  
  if Idx < 0 then
  begin
    Result := 'no_otp';
    Exit;
  end;
  
  Rec := FRecords[Idx];
  
  if Rec^.IsVerified then
    Result := 'verified'
  else if Now > Rec^.ExpiresAt then
    Result := 'expired'
  else if Rec^.Attempts >= FConfig.MaxAttempts then
    Result := 'locked'
  else
  begin
    MinLeft := Round((Rec^.ExpiresAt - Now) * 24 * 60);
    Result := Format('active_%d_min_left', [Max(0, MinLeft)]);
  end;
end;

procedure TOTPManager.InvalidateOTP(const APhone: string);
var
  Idx: Integer;
  Rec: ^TOTPRecord;
begin
  Idx := FindRecord(APhone);
  if Idx >= 0 then
  begin
    Rec := FRecords[Idx];
    Dispose(Rec);
    FRecords.Delete(Idx);
    WriteLn('ยกเลิก OTP: ', APhone);
  end;
end;

function TOTPManager.GetActiveCount: Integer;
var
  i: Integer;
  Rec: ^TOTPRecord;
begin
  Result := 0;
  CleanExpired;
  
  for i := 0 to FRecords.Count - 1 do
  begin
    Rec := FRecords[i];
    if (Now <= Rec^.ExpiresAt) and not Rec^.IsVerified then
      Inc(Result);
  end;
end;

function TOTPManager.GetStats: string;
var
  Active, Verified, Expired, Locked: Integer;
  i: Integer;
  Rec: ^TOTPRecord;
begin
  Active := 0; Verified := 0; Expired := 0; Locked := 0;
  
  for i := 0 to FRecords.Count - 1 do
  begin
    Rec := FRecords[i];
    if Rec^.IsVerified then Inc(Verified)
    else if Now > Rec^.ExpiresAt then Inc(Expired)
    else if Rec^.Attempts >= FConfig.MaxAttempts then Inc(Locked)
    else Inc(Active);
  end;
  
  Result := Format('Active=%d, Verified=%d, Expired=%d, Locked=%d',
    [Active, Verified, Expired, Locked]);
end;

end.
```

---

## 69.4 ตัวอย่างโปรแกรม OTP Login

```pascal
program otp_login_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  sms_twilio, otp_system;

type
  TLoginSystem = class
  private
    FSMS: TTwilioSMS;
    FOTP: TOTPManager;
    FUsers: TStringList;  // phone -> username
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure RegisterUser(const APhone, AUsername: string);
    function Login(const APhone: string): Boolean;
    function VerifyLogin(const APhone, AOTP: string): Boolean;
    procedure Logout(const APhone: string);
    function IsLoggedIn(const APhone: string): Boolean;
  end;

var
  LoggedIn: TStringList;  // โปรดักชั่นใช้ Session storage จริง

constructor TLoginSystem.Create;
var
  Config: TOTPConfig;
begin
  inherited Create;
  FUsers := TStringList.Create;
  LoggedIn := TStringList.Create;
  
  // ตั้งค่า Twilio
  FSMS := TTwilioSMS.Create(
    'YOUR_ACCOUNT_SID',
    'YOUR_AUTH_TOKEN',
    '+15017122661'
  );
  
  // ตั้งค่า OTP
  Config := DefaultOTPConfig;
  Config.Length := 6;
  Config.ExpirationMinutes := 5;
  Config.MessageTemplate := '[MyApp] รหัสยืนยัน: {OTP} (หมดอายุใน {EXPIRE} นาที)';
  
  FOTP := TOTPManager.Create(FSMS, Config);
end;

destructor TLoginSystem.Destroy;
begin
  FOTP.Free;
  FSMS.Free;
  FUsers.Free;
  LoggedIn.Free;
  inherited Destroy;
end;

procedure TLoginSystem.RegisterUser(const APhone, AUsername: string);
begin
  FUsers.Values[APhone] := AUsername;
  WriteLn('ลงทะเบียน: ', AUsername, ' (', APhone, ')');
end;

function TLoginSystem.Login(const APhone: string): Boolean;
begin
  Result := False;
  
  // ตรวจสอบว่ามีผู้ใช้
  if FUsers.IndexOfName(APhone) < 0 then
  begin
    WriteLn('ไม่พบผู้ใช้: ', APhone);
    Exit;
  end;
  
  WriteLn('กำลังส่ง OTP ไปยัง: ', APhone, '...');
  Result := FOTP.SendOTP(APhone, 'login');
  
  if Result then
    WriteLn('โปรดกรอกรหัส OTP ที่ได้รับทาง SMS')
  else
    WriteLn('ส่ง OTP ล้มเหลว กรุณาลองใหม่');
end;

function TLoginSystem.VerifyLogin(const APhone, AOTP: string): Boolean;
var
  Username: string;
begin
  Result := FOTP.VerifyOTP(APhone, AOTP);
  
  if Result then
  begin
    Username := FUsers.Values[APhone];
    LoggedIn.Values[APhone] := FormatDateTime('yyyy-mm-dd hh:nn:ss', Now);
    WriteLn('เข้าสู่ระบบสำเร็จ: ', Username);
    WriteLn('เวลาเข้าสู่ระบบ: ', FormatDateTime('dd/mm/yyyy hh:nn:ss', Now));
  end
  else
    WriteLn('OTP ไม่ถูกต้อง');
end;

function TLoginSystem.IsLoggedIn(const APhone: string): Boolean;
begin
  Result := LoggedIn.IndexOfName(APhone) >= 0;
end;

procedure TLoginSystem.Logout(const APhone: string);
var
  Idx: Integer;
begin
  Idx := LoggedIn.IndexOfName(APhone);
  if Idx >= 0 then
  begin
    LoggedIn.Delete(Idx);
    WriteLn('ออกจากระบบ: ', FUsers.Values[APhone]);
  end;
end;

// Demo
var
  LoginSys: TLoginSystem;
  Phone, OTP: string;
begin
  WriteLn('=== OTP Login System Demo ===');
  
  LoginSys := TLoginSystem.Create;
  try
    // ลงทะเบียนผู้ใช้
    LoginSys.RegisterUser('+66812345678', 'สมชาย ใจดี');
    LoginSys.RegisterUser('+66898765432', 'สมหญิง รักเรียน');
    
    WriteLn('');
    Write('ใส่เบอร์โทร (หรือ q เพื่อออก): ');
    ReadLn(Phone);
    
    if Phone = 'q' then Exit;
    
    if LoginSys.Login(Phone) then
    begin
      Write('ใส่รหัส OTP: ');
      ReadLn(OTP);
      
      if LoginSys.VerifyLogin(Phone, OTP) then
      begin
        WriteLn('ยินดีต้อนรับ! เข้าสู่ระบบสำเร็จ');
        
        Write('กด Enter เพื่อออกจากระบบ...');
        ReadLn;
        
        LoginSys.Logout(Phone);
      end
      else
        WriteLn('ล็อกอินล้มเหลว');
    end;
    
  finally
    LoginSys.Free;
  end;
end.
```

---

## 69.5 SMS via Modem

```pascal
unit sms_modem;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  {$IFDEF WINDOWS}
  Windows;
  {$ELSE}
  BaseUnix;
  {$ENDIF}

type
  TATCommand = class
  private
    FPort: THandle;
    FPortName: string;
    FBaudRate: Integer;
    FTimeout: Integer;
    
    function OpenPort: Boolean;
    procedure ClosePort;
    function SendATCommand(const ACommand: string): string;
    function WaitForResponse(const AExpected: string; ATimeout: Integer): Boolean;
    
  public
    constructor Create(const APortName: string; ABaudRate: Integer = 9600);
    destructor Destroy; override;
    
    function Connect: Boolean;
    procedure Disconnect;
    
    // AT Commands
    function CheckModem: Boolean;
    function GetSignalStrength: Integer;
    function GetOperatorName: string;
    function SendSMS(const APhone, AMessage: string): Boolean;
    function ReadSMS(AIndex: Integer; out APhone, AMessage: string): Boolean;
    function DeleteSMS(AIndex: Integer): Boolean;
    function GetSMSCount: Integer;
    
    property PortName: string read FPortName;
  end;

implementation

constructor TATCommand.Create(const APortName: string; ABaudRate: Integer);
begin
  inherited Create;
  FPortName := APortName;
  FBaudRate := ABaudRate;
  FTimeout := 5000;
  FPort := INVALID_HANDLE_VALUE;
end;

destructor TATCommand.Destroy;
begin
  Disconnect;
  inherited Destroy;
end;

function TATCommand.OpenPort: Boolean;
{$IFDEF WINDOWS}
var
  DCB: TDCB;
  CommTimeouts: TCommTimeouts;
begin
  Result := False;
  
  FPort := CreateFile(PChar('\\.\' + FPortName),
    GENERIC_READ or GENERIC_WRITE,
    0, nil, OPEN_EXISTING,
    FILE_ATTRIBUTE_NORMAL, 0);
    
  if FPort = INVALID_HANDLE_VALUE then
  begin
    WriteLn('ไม่สามารถเปิด port: ', FPortName);
    Exit;
  end;
  
  // ตั้งค่า baud rate
  GetCommState(FPort, DCB);
  DCB.BaudRate := FBaudRate;
  DCB.ByteSize := 8;
  DCB.StopBits := ONESTOPBIT;
  DCB.Parity := NOPARITY;
  SetCommState(FPort, DCB);
  
  // ตั้งค่า timeout
  CommTimeouts.ReadIntervalTimeout := 50;
  CommTimeouts.ReadTotalTimeoutMultiplier := 10;
  CommTimeouts.ReadTotalTimeoutConstant := 2000;
  CommTimeouts.WriteTotalTimeoutMultiplier := 10;
  CommTimeouts.WriteTotalTimeoutConstant := 2000;
  SetCommTimeouts(FPort, CommTimeouts);
  
  Result := True;
end;
{$ELSE}
begin
  // Linux implementation
  Result := False;
  WriteLn('Linux serial port: ', FPortName);
  // ใช้ fpOpen สำหรับ Linux
end;
{$ENDIF}

procedure TATCommand.ClosePort;
begin
  {$IFDEF WINDOWS}
  if FPort <> INVALID_HANDLE_VALUE then
  begin
    CloseHandle(FPort);
    FPort := INVALID_HANDLE_VALUE;
  end;
  {$ENDIF}
end;

function TATCommand.SendATCommand(const ACommand: string): string;
var
  Cmd: string;
  Buffer: array[0..255] of AnsiChar;
  BytesWritten, BytesRead: DWORD;
  Response: string;
  StartTime: Cardinal;
begin
  Result := '';
  Cmd := ACommand + #13#10;
  
  {$IFDEF WINDOWS}
  WriteFile(FPort, Cmd[1], Length(Cmd), BytesWritten, nil);
  
  Response := '';
  StartTime := GetTickCount;
  
  while GetTickCount - StartTime < FTimeout do
  begin
    FillChar(Buffer, SizeOf(Buffer), 0);
    ReadFile(FPort, Buffer, SizeOf(Buffer) - 1, BytesRead, nil);
    
    if BytesRead > 0 then
    begin
      Response := Response + String(Buffer);
      
      // ตรวจสอบว่าได้รับ response สมบูรณ์
      if (Pos('OK', Response) > 0) or 
         (Pos('ERROR', Response) > 0) or
         (Pos('+CMGS:', Response) > 0) then
      begin
        Break;
      end;
    end;
    
    Sleep(100);
  end;
  
  Result := Trim(Response);
  {$ENDIF}
end;

function TATCommand.Connect: Boolean;
begin
  Result := OpenPort;
  if Result then
  begin
    // Initialize modem
    SendATCommand('ATZ');  // Reset
    Sleep(500);
    var Resp := SendATCommand('AT');  // Check
    Result := Pos('OK', Resp) > 0;
    
    if Result then
    begin
      SendATCommand('ATE0');  // Echo off
      SendATCommand('AT+CMGF=1');  // SMS text mode
      WriteLn('Modem connected: ', FPortName);
    end;
  end;
end;

procedure TATCommand.Disconnect;
begin
  if FPort <> INVALID_HANDLE_VALUE then
  begin
    SendATCommand('ATZ');
    ClosePort;
    WriteLn('Modem disconnected');
  end;
end;

function TATCommand.CheckModem: Boolean;
var
  Resp: string;
begin
  Resp := SendATCommand('AT');
  Result := Pos('OK', Resp) > 0;
end;

function TATCommand.GetSignalStrength: Integer;
var
  Resp: string;
  P1, P2: Integer;
begin
  Result := -1;
  Resp := SendATCommand('AT+CSQ');
  
  // Response: +CSQ: 20,0
  P1 := Pos('+CSQ: ', Resp);
  if P1 > 0 then
  begin
    P1 := P1 + 6;
    P2 := Pos(',', Resp, P1);
    if P2 > P1 then
      Result := StrToIntDef(Copy(Resp, P1, P2 - P1), -1);
  end;
end;

function TATCommand.GetOperatorName: string;
var
  Resp: string;
  P1, P2: Integer;
begin
  Result := '';
  Resp := SendATCommand('AT+COPS?');
  
  // Response: +COPS: 0,0,"AIS",7
  P1 := Pos('"', Resp);
  if P1 > 0 then
  begin
    Inc(P1);
    P2 := Pos('"', Resp, P1);
    if P2 > P1 then
      Result := Copy(Resp, P1, P2 - P1);
  end;
end;

function TATCommand.SendSMS(const APhone, AMessage: string): Boolean;
var
  Resp: string;
begin
  Result := False;
  
  // ตั้งค่า recipient
  SendATCommand('AT+CMGF=1');  // Text mode
  Resp := SendATCommand('AT+CMGS="' + APhone + '"');
  
  if Pos('>', Resp) > 0 then
  begin
    // ส่งข้อความและ Ctrl+Z
    Resp := SendATCommand(AMessage + #26);
    Result := Pos('+CMGS:', Resp) > 0;
    
    if Result then
      WriteLn('SMS ส่งสำเร็จ via modem')
    else
      WriteLn('SMS ส่งล้มเหลว: ', Resp);
  end
  else
    WriteLn('ไม่ได้รับ prompt ">" จาก modem');
end;

function TATCommand.ReadSMS(AIndex: Integer; out APhone, AMessage: string): Boolean;
var
  Resp: string;
  Lines: TStringList;
begin
  Result := False;
  APhone := '';
  AMessage := '';
  
  Resp := SendATCommand(Format('AT+CMGR=%d', [AIndex]));
  
  Lines := TStringList.Create;
  try
    Lines.Text := Resp;
    
    // บรรทัดแรก: +CMGR: "REC READ","+66812345678",,"24/01/01,12:00:00+28"
    if Lines.Count >= 2 then
    begin
      var Line1 := Lines[0];
      var P1 := Pos('","+', Line1);  // หา phone number
      if P1 > 0 then
      begin
        P1 := P1 + 3;  // ข้าม ',"+'
        var P2 := Pos('"', Line1, P1);
        if P2 > P1 then
        begin
          APhone := '+' + Copy(Line1, P1, P2 - P1);
          AMessage := Lines[1];
          Result := True;
        end;
      end;
    end;
  finally
    Lines.Free;
  end;
end;

function TATCommand.DeleteSMS(AIndex: Integer): Boolean;
var
  Resp: string;
begin
  Resp := SendATCommand(Format('AT+CMGD=%d', [AIndex]));
  Result := Pos('OK', Resp) > 0;
end;

function TATCommand.GetSMSCount: Integer;
var
  Resp: string;
  P1, P2: Integer;
begin
  Result := 0;
  Resp := SendATCommand('AT+CPMS?');
  
  // Response: +CPMS: "SM",5,20,"SM",5,20,"SM",5,20
  P1 := Pos('"SM",', Resp);
  if P1 > 0 then
  begin
    P1 := P1 + 5;
    P2 := Pos(',', Resp, P1);
    if P2 > P1 then
      Result := StrToIntDef(Copy(Resp, P1, P2 - P1), 0);
  end;
end;

end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Twilio SMS API** - ส่ง SMS ผ่าน REST API
2. **Thai SMS Gateways** - ใช้ gateway ท้องถิ่น
3. **OTP System** - สร้างระบบ OTP สมบูรณ์
4. **SMS via Modem** - ส่ง SMS ผ่าน GSM modem
5. **OTP Login** - ระบบ login ด้วย OTP

SMS เป็นช่องทางที่เชื่อถือได้สำหรับ OTP และ notifications สำคัญ เพราะผู้ใช้ส่วนใหญ่มีโทรศัพท์มือถือและตรวจสอบ SMS อย่างสม่ำเสมอ
