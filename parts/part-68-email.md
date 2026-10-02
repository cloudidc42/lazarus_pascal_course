# ตอนที่ 68: Email Programming ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการส่ง email ผ่าน SMTP, การอ่าน email ผ่าน IMAP/POP3, การส่ง attachments และ HTML email โดยใช้ Synapse library

---

## 68.1 ติดตั้ง Synapse

```bash
# ติดตั้ง Synapse ผ่าน OPM หรือ manual
# https://synapse.ararat.cz/

# ใน Package ของโปรเจค ต้องเพิ่ม:
uses
  smtpsend,    # SMTP client
  pop3send,    # POP3 client
  imapsend,    # IMAP client
  mimemess,    # MIME message
  mimepart,    # MIME part
  ssl_openssl; # SSL support
```

---

## 68.2 SMTP Client

```pascal
unit email_smtp;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  smtpsend, mimemess, mimepart,
  ssl_openssl;

type
  TEmailAttachment = record
    FileName: string;
    MimeType: string;
    Data: TStream;
  end;

  TEmailMessage = class
  private
    FFrom: string;
    FFromName: string;
    FTo: TStringList;
    FCC: TStringList;
    FBCC: TStringList;
    FReplyTo: string;
    FSubject: string;
    FBody: string;
    FHTMLBody: string;
    FAttachments: TList;
    FPriority: Integer;
    FHeaders: TStringList;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure AddTo(const AEmail: string);
    procedure AddCC(const AEmail: string);
    procedure AddBCC(const AEmail: string);
    procedure AddAttachment(const AFileName: string); overload;
    procedure AddAttachment(const AFileName: string; AStream: TStream); overload;
    procedure AddHeader(const AName, AValue: string);
    
    property From: string read FFrom write FFrom;
    property FromName: string read FFromName write FFromName;
    property Subject: string read FSubject write FSubject;
    property Body: string read FBody write FBody;
    property HTMLBody: string read FHTMLBody write FHTMLBody;
    property Priority: Integer read FPriority write FPriority;
    property ToList: TStringList read FTo;
    property CCList: TStringList read FCC;
  end;

  TSMTPConfig = record
    Host: string;
    Port: Integer;
    Username: string;
    Password: string;
    UseSSL: Boolean;
    UseTLS: Boolean;
    Timeout: Integer;
  end;

  TSMTPClient = class
  private
    FSMTP: TSMTPSend;
    FConfig: TSMTPConfig;
    FLastError: string;
    
    function BuildMimeMessage(AMessage: TEmailMessage): string;
    
  public
    constructor Create(const AConfig: TSMTPConfig);
    destructor Destroy; override;
    
    function Connect: Boolean;
    procedure Disconnect;
    function SendMessage(AMessage: TEmailMessage): Boolean;
    function SendSimple(const AFrom, ATo, ASubject, ABody: string): Boolean;
    
    property Config: TSMTPConfig read FConfig write FConfig;
    property LastError: string read FLastError;
  end;

  // Predefined configs
  function GmailConfig(const AEmail, APassword: string): TSMTPConfig;
  function OutlookConfig(const AEmail, APassword: string): TSMTPConfig;
  function CustomConfig(const AHost: string; APort: Integer;
    const AUser, APassword: string; ASSL: Boolean = True): TSMTPConfig;

implementation

uses
  base64;

function GmailConfig(const AEmail, APassword: string): TSMTPConfig;
begin
  Result.Host := 'smtp.gmail.com';
  Result.Port := 587;
  Result.Username := AEmail;
  Result.Password := APassword;
  Result.UseSSL := False;
  Result.UseTLS := True;
  Result.Timeout := 30000;
end;

function OutlookConfig(const AEmail, APassword: string): TSMTPConfig;
begin
  Result.Host := 'smtp-mail.outlook.com';
  Result.Port := 587;
  Result.Username := AEmail;
  Result.Password := APassword;
  Result.UseSSL := False;
  Result.UseTLS := True;
  Result.Timeout := 30000;
end;

function CustomConfig(const AHost: string; APort: Integer;
  const AUser, APassword: string; ASSL: Boolean): TSMTPConfig;
begin
  Result.Host := AHost;
  Result.Port := APort;
  Result.Username := AUser;
  Result.Password := APassword;
  Result.UseSSL := ASSL;
  Result.UseTLS := not ASSL;
  Result.Timeout := 30000;
end;

{ TEmailMessage }

constructor TEmailMessage.Create;
begin
  inherited Create;
  FTo := TStringList.Create;
  FCC := TStringList.Create;
  FBCC := TStringList.Create;
  FAttachments := TList.Create;
  FHeaders := TStringList.Create;
  FPriority := 3;  // Normal
end;

destructor TEmailMessage.Destroy;
var
  i: Integer;
  Att: ^TEmailAttachment;
begin
  for i := 0 to FAttachments.Count - 1 do
  begin
    Att := FAttachments[i];
    if Assigned(Att^.Data) then
      Att^.Data.Free;
    Dispose(Att);
  end;
  
  FAttachments.Free;
  FHeaders.Free;
  FBCC.Free;
  FCC.Free;
  FTo.Free;
  inherited Destroy;
end;

procedure TEmailMessage.AddTo(const AEmail: string);
begin
  FTo.Add(AEmail);
end;

procedure TEmailMessage.AddCC(const AEmail: string);
begin
  FCC.Add(AEmail);
end;

procedure TEmailMessage.AddBCC(const AEmail: string);
begin
  FBCC.Add(AEmail);
end;

procedure TEmailMessage.AddAttachment(const AFileName: string);
var
  Stream: TFileStream;
  Att: ^TEmailAttachment;
begin
  if not FileExists(AFileName) then Exit;
  
  Stream := TFileStream.Create(AFileName, fmOpenRead or fmShareDenyNone);
  
  New(Att);
  Att^.FileName := ExtractFileName(AFileName);
  Att^.MimeType := 'application/octet-stream';  // default
  Att^.Data := Stream;
  FAttachments.Add(Att);
end;

procedure TEmailMessage.AddAttachment(const AFileName: string; AStream: TStream);
var
  Att: ^TEmailAttachment;
begin
  New(Att);
  Att^.FileName := AFileName;
  Att^.MimeType := 'application/octet-stream';
  Att^.Data := AStream;
  FAttachments.Add(Att);
end;

procedure TEmailMessage.AddHeader(const AName, AValue: string);
begin
  FHeaders.Add(AName + ': ' + AValue);
end;

{ TSMTPClient }

constructor TSMTPClient.Create(const AConfig: TSMTPConfig);
begin
  inherited Create;
  FConfig := AConfig;
  FSMTP := TSMTPSend.Create;
  FSMTP.Timeout := AConfig.Timeout;
end;

destructor TSMTPClient.Destroy;
begin
  FSMTP.Free;
  inherited Destroy;
end;

function TSMTPClient.Connect: Boolean;
begin
  Result := False;
  
  FSMTP.TargetHost := FConfig.Host;
  FSMTP.TargetPort := IntToStr(FConfig.Port);
  FSMTP.UserName := FConfig.Username;
  FSMTP.Password := FConfig.Password;
  
  if FConfig.UseSSL then
    FSMTP.SSLType := LT_SSLv23
  else
    FSMTP.SSLType := LT_noTLS;
    
  // Connect
  if not FSMTP.Connect then
  begin
    FLastError := 'ไม่สามารถเชื่อมต่อกับ ' + FConfig.Host;
    Exit;
  end;
  
  // EHLO
  if not FSMTP.EhloCmd then
  begin
    FLastError := 'EHLO ล้มเหลว';
    FSMTP.Disconnect;
    Exit;
  end;
  
  // STARTTLS
  if FConfig.UseTLS then
  begin
    if not FSMTP.StartTLS then
    begin
      FLastError := 'STARTTLS ล้มเหลว';
      FSMTP.Disconnect;
      Exit;
    end;
    
    if not FSMTP.EhloCmd then
    begin
      FLastError := 'EHLO หลัง TLS ล้มเหลว';
      FSMTP.Disconnect;
      Exit;
    end;
  end;
  
  // AUTH
  if (FConfig.Username <> '') and (FConfig.Password <> '') then
  begin
    if not FSMTP.Auth then
    begin
      FLastError := 'Authentication ล้มเหลว';
      FSMTP.Disconnect;
      Exit;
    end;
  end;
  
  Result := True;
end;

procedure TSMTPClient.Disconnect;
begin
  if Assigned(FSMTP) then
    FSMTP.Disconnect;
end;

function TSMTPClient.BuildMimeMessage(AMessage: TEmailMessage): string;
var
  Mime: TMimeMess;
  TextPart, HTMLPart, MixedPart, AlternPart: TMimePart;
  AttPart: TMimePart;
  i: Integer;
  Att: ^TEmailAttachment;
  AttData: TStringList;
  MS: TMemoryStream;
begin
  Result := '';
  Mime := TMimeMess.Create;
  try
    // Set headers
    Mime.Header.From := AMessage.From;
    if AMessage.FromName <> '' then
      Mime.Header.From := '"' + AMessage.FromName + '" <' + AMessage.From + '>';
    Mime.Header.ToList.Text := AMessage.ToList.Text;
    if AMessage.CCList.Count > 0 then
      Mime.Header.CCList.Text := AMessage.CCList.Text;
    Mime.Header.Subject := AMessage.Subject;
    Mime.Header.Date := Now;
    
    // Custom headers
    for i := 0 to AMessage.FHeaders.Count - 1 do
      Mime.Header.CustomHeaders.Add(AMessage.FHeaders[i]);
    
    // Determine content type
    if AMessage.FAttachments.Count > 0 then
    begin
      // Mixed (with attachments)
      MixedPart := Mime.AddPart(nil);
      MixedPart.PrimaryType := 'multipart';
      MixedPart.SecondaryType := 'mixed';
      
      if (AMessage.Body <> '') and (AMessage.HTMLBody <> '') then
      begin
        // Alternative inside mixed
        AlternPart := Mime.AddPart(MixedPart);
        AlternPart.PrimaryType := 'multipart';
        AlternPart.SecondaryType := 'alternative';
        
        TextPart := Mime.AddPart(AlternPart);
        TextPart.PrimaryType := 'text';
        TextPart.SecondaryType := 'plain';
        TextPart.Charset := 'utf-8';
        TextPart.AddHeaderField('Content-Transfer-Encoding', 'quoted-printable');
        TextPart.DecodedLines.Text := AMessage.Body;
        
        HTMLPart := Mime.AddPart(AlternPart);
        HTMLPart.PrimaryType := 'text';
        HTMLPart.SecondaryType := 'html';
        HTMLPart.Charset := 'utf-8';
        HTMLPart.DecodedLines.Text := AMessage.HTMLBody;
      end
      else if AMessage.HTMLBody <> '' then
      begin
        HTMLPart := Mime.AddPart(MixedPart);
        HTMLPart.PrimaryType := 'text';
        HTMLPart.SecondaryType := 'html';
        HTMLPart.Charset := 'utf-8';
        HTMLPart.DecodedLines.Text := AMessage.HTMLBody;
      end
      else
      begin
        TextPart := Mime.AddPart(MixedPart);
        TextPart.PrimaryType := 'text';
        TextPart.SecondaryType := 'plain';
        TextPart.Charset := 'utf-8';
        TextPart.DecodedLines.Text := AMessage.Body;
      end;
      
      // Attachments
      for i := 0 to AMessage.FAttachments.Count - 1 do
      begin
        Att := AMessage.FAttachments[i];
        AttPart := Mime.AddPart(MixedPart);
        AttPart.PrimaryType := 'application';
        AttPart.SecondaryType := 'octet-stream';
        AttPart.Disposition := 'attachment';
        AttPart.FileName := Att^.FileName;
        AttPart.EncodingType := ME_BASE64;
        
        // อ่านข้อมูล
        Att^.Data.Position := 0;
        MS := TMemoryStream.Create;
        try
          MS.CopyFrom(Att^.Data, Att^.Data.Size);
          MS.Position := 0;
          AttPart.DecodedLines.LoadFromStream(MS);
        finally
          MS.Free;
        end;
      end;
    end
    else if (AMessage.Body <> '') and (AMessage.HTMLBody <> '') then
    begin
      // Alternative (text + HTML, no attachments)
      AlternPart := Mime.AddPart(nil);
      AlternPart.PrimaryType := 'multipart';
      AlternPart.SecondaryType := 'alternative';
      
      TextPart := Mime.AddPart(AlternPart);
      TextPart.PrimaryType := 'text';
      TextPart.SecondaryType := 'plain';
      TextPart.Charset := 'utf-8';
      TextPart.DecodedLines.Text := AMessage.Body;
      
      HTMLPart := Mime.AddPart(AlternPart);
      HTMLPart.PrimaryType := 'text';
      HTMLPart.SecondaryType := 'html';
      HTMLPart.Charset := 'utf-8';
      HTMLPart.DecodedLines.Text := AMessage.HTMLBody;
    end
    else
    begin
      // Simple text or HTML
      TextPart := Mime.AddPart(nil);
      if AMessage.HTMLBody <> '' then
      begin
        TextPart.PrimaryType := 'text';
        TextPart.SecondaryType := 'html';
        TextPart.DecodedLines.Text := AMessage.HTMLBody;
      end
      else
      begin
        TextPart.PrimaryType := 'text';
        TextPart.SecondaryType := 'plain';
        TextPart.DecodedLines.Text := AMessage.Body;
      end;
      TextPart.Charset := 'utf-8';
    end;
    
    // Encode
    Mime.EncodeMessage;
    Result := Mime.Lines.Text;
    
  finally
    Mime.Free;
  end;
end;

function TSMTPClient.SendMessage(AMessage: TEmailMessage): Boolean;
var
  MimeData: string;
  i: Integer;
begin
  Result := False;
  FLastError := '';
  
  if AMessage.ToList.Count = 0 then
  begin
    FLastError := 'ไม่มีผู้รับ';
    Exit;
  end;
  
  if not Connect then Exit;
  
  try
    // MAIL FROM
    if not FSMTP.MailFrom(AMessage.From, 0) then
    begin
      FLastError := 'MAIL FROM ล้มเหลว: ' + FSMTP.ResultString;
      Exit;
    end;
    
    // RCPT TO
    for i := 0 to AMessage.ToList.Count - 1 do
    begin
      if not FSMTP.MailTo(AMessage.ToList[i]) then
      begin
        FLastError := 'RCPT TO ล้มเหลว สำหรับ: ' + AMessage.ToList[i];
        Exit;
      end;
    end;
    
    // CC
    for i := 0 to AMessage.CCList.Count - 1 do
      FSMTP.MailTo(AMessage.CCList[i]);
      
    // DATA
    MimeData := BuildMimeMessage(AMessage);
    
    if not FSMTP.MailData(MimeData) then
    begin
      FLastError := 'ส่ง DATA ล้มเหลว: ' + FSMTP.ResultString;
      Exit;
    end;
    
    Result := True;
    WriteLn('ส่ง email สำเร็จ: ', AMessage.Subject);
    
  finally
    Disconnect;
  end;
end;

function TSMTPClient.SendSimple(const AFrom, ATo, ASubject, ABody: string): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := AFrom;
    Msg.AddTo(ATo);
    Msg.Subject := ASubject;
    Msg.Body := ABody;
    Result := SendMessage(Msg);
  finally
    Msg.Free;
  end;
end;

end.
```

---

## 68.3 HTML Email Templates

```pascal
unit email_templates;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TEmailTemplate = class
  private
    FTemplatePath: string;
    FVariables: TStringList;
    
  public
    constructor Create(const ATemplatePath: string = '');
    destructor Destroy; override;
    
    procedure SetVariable(const AName, AValue: string);
    function Render(const ATemplateText: string = ''): string;
    function RenderFile(const AFileName: string): string;
    
    property Variables: TStringList read FVariables;
  end;

  // Templates สำเร็จรูป
  function WelcomeEmailHTML(const AName, AActivationLink: string): string;
  function OrderConfirmHTML(const AOrderNo, ACustomerName: string; 
    ATotal: Double): string;
  function PasswordResetHTML(const AName, AResetLink: string): string;
  function NotificationHTML(const ATitle, AMessage, AActionLink, 
    AActionText: string): string;

implementation

{ TEmailTemplate }

constructor TEmailTemplate.Create(const ATemplatePath: string);
begin
  inherited Create;
  FTemplatePath := ATemplatePath;
  FVariables := TStringList.Create;
end;

destructor TEmailTemplate.Destroy;
begin
  FVariables.Free;
  inherited Destroy;
end;

procedure TEmailTemplate.SetVariable(const AName, AValue: string);
begin
  FVariables.Values[AName] := AValue;
end;

function TEmailTemplate.Render(const ATemplateText: string): string;
var
  i: Integer;
  Placeholder: string;
begin
  Result := ATemplateText;
  
  for i := 0 to FVariables.Count - 1 do
  begin
    Placeholder := '{{' + FVariables.Names[i] + '}}';
    Result := StringReplace(Result, Placeholder, 
      FVariables.ValueFromIndex[i], [rfReplaceAll]);
  end;
end;

function TEmailTemplate.RenderFile(const AFileName: string): string;
var
  Template: TStringList;
begin
  Template := TStringList.Create;
  try
    Template.LoadFromFile(AFileName);
    Result := Render(Template.Text);
  finally
    Template.Free;
  end;
end;

// Base HTML template
function BaseEmailHTML(const AContent, ACompanyName: string): string;
begin
  Result :=
    '<!DOCTYPE html>' + LineEnding +
    '<html lang="th">' + LineEnding +
    '<head>' + LineEnding +
    '  <meta charset="UTF-8">' + LineEnding +
    '  <meta name="viewport" content="width=device-width, initial-scale=1.0">' + LineEnding +
    '  <style>' + LineEnding +
    '    body { font-family: "Sarabun", Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 0; }' + LineEnding +
    '    .container { max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; }' + LineEnding +
    '    .header { background: #2c3e50; color: white; padding: 30px; text-align: center; }' + LineEnding +
    '    .header h1 { margin: 0; font-size: 24px; }' + LineEnding +
    '    .content { padding: 30px; color: #333; }' + LineEnding +
    '    .footer { background: #ecf0f1; padding: 20px; text-align: center; color: #666; font-size: 12px; }' + LineEnding +
    '    .btn { display: inline-block; padding: 12px 24px; background: #3498db; color: white; text-decoration: none; border-radius: 4px; margin: 15px 0; }' + LineEnding +
    '    .btn:hover { background: #2980b9; }' + LineEnding +
    '    .success { color: #27ae60; }' + LineEnding +
    '    .warning { color: #e67e22; }' + LineEnding +
    '    .error { color: #e74c3c; }' + LineEnding +
    '    table { width: 100%; border-collapse: collapse; }' + LineEnding +
    '    th { background: #2c3e50; color: white; padding: 10px; text-align: left; }' + LineEnding +
    '    td { padding: 8px 10px; border-bottom: 1px solid #eee; }' + LineEnding +
    '    tr:last-child td { border-bottom: none; }' + LineEnding +
    '  </style>' + LineEnding +
    '</head>' + LineEnding +
    '<body>' + LineEnding +
    '  <div class="container">' + LineEnding +
    '    <div class="header"><h1>' + ACompanyName + '</h1></div>' + LineEnding +
    '    <div class="content">' + LineEnding +
    AContent +
    '    </div>' + LineEnding +
    '    <div class="footer">' + LineEnding +
    '      <p>© ' + IntToStr(YearOf(Now)) + ' ' + ACompanyName + '. All rights reserved.</p>' + LineEnding +
    '      <p>หากคุณไม่ต้องการรับอีเมลนี้ <a href="#">ยกเลิกการสมัคร</a></p>' + LineEnding +
    '    </div>' + LineEnding +
    '  </div>' + LineEnding +
    '</body>' + LineEnding +
    '</html>';
end;

function WelcomeEmailHTML(const AName, AActivationLink: string): string;
var
  Content: string;
begin
  Content :=
    '<h2>ยินดีต้อนรับ, ' + AName + '!</h2>' + LineEnding +
    '<p>ขอบคุณที่สมัครสมาชิกกับเรา เพียงคลิกปุ่มด้านล่างเพื่อยืนยันบัญชีของคุณ</p>' + LineEnding +
    '<p><a class="btn" href="' + AActivationLink + '">ยืนยันบัญชี</a></p>' + LineEnding +
    '<p>หรือคัดลอก URL นี้ไปยังเบราว์เซอร์:</p>' + LineEnding +
    '<p style="word-break: break-all; color: #666;">' + AActivationLink + '</p>' + LineEnding +
    '<p><small>ลิงก์นี้จะหมดอายุใน 24 ชั่วโมง</small></p>';
    
  Result := BaseEmailHTML(Content, 'MyApp');
end;

function OrderConfirmHTML(const AOrderNo, ACustomerName: string; 
  ATotal: Double): string;
var
  Content: string;
begin
  Content :=
    '<h2>ยืนยันคำสั่งซื้อ</h2>' + LineEnding +
    '<p>เรียน ' + ACustomerName + ',</p>' + LineEnding +
    '<p>เราได้รับคำสั่งซื้อของคุณแล้ว</p>' + LineEnding +
    '<table>' + LineEnding +
    '<tr><th>รายละเอียด</th><th>ข้อมูล</th></tr>' + LineEnding +
    '<tr><td>เลขที่คำสั่งซื้อ</td><td><strong>' + AOrderNo + '</strong></td></tr>' + LineEnding +
    '<tr><td>วันที่</td><td>' + FormatDateTime('dd/mm/yyyy hh:nn', Now) + '</td></tr>' + LineEnding +
    '<tr><td>ยอดรวม</td><td><strong class="success">฿' + FormatFloat('#,##0.00', ATotal) + '</strong></td></tr>' + LineEnding +
    '</table>' + LineEnding +
    '<p>เราจะแจ้งให้ทราบเมื่อสินค้าถูกจัดส่ง</p>' + LineEnding +
    '<p><a class="btn" href="#">ติดตามคำสั่งซื้อ</a></p>';
    
  Result := BaseEmailHTML(Content, 'MyShop');
end;

function PasswordResetHTML(const AName, AResetLink: string): string;
var
  Content: string;
begin
  Content :=
    '<h2>รีเซ็ตรหัสผ่าน</h2>' + LineEnding +
    '<p>เรียน ' + AName + ',</p>' + LineEnding +
    '<p>เราได้รับคำขอรีเซ็ตรหัสผ่านสำหรับบัญชีของคุณ</p>' + LineEnding +
    '<p><a class="btn" href="' + AResetLink + '">รีเซ็ตรหัสผ่าน</a></p>' + LineEnding +
    '<p class="warning">ลิงก์นี้จะหมดอายุใน 1 ชั่วโมง</p>' + LineEnding +
    '<p>หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน กรุณาละเว้นอีเมลนี้</p>';
    
  Result := BaseEmailHTML(Content, 'MyApp');
end;

function NotificationHTML(const ATitle, AMessage, AActionLink, AActionText: string): string;
var
  Content: string;
begin
  Content :=
    '<h2>' + ATitle + '</h2>' + LineEnding +
    '<p>' + AMessage + '</p>';
    
  if (AActionLink <> '') and (AActionText <> '') then
    Content := Content + 
      '<p><a class="btn" href="' + AActionLink + '">' + AActionText + '</a></p>';
      
  Result := BaseEmailHTML(Content, 'MyApp');
end;

end.
```

---

## 68.4 ระบบแจ้งเตือน Email

```pascal
unit notification_system;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils,
  email_smtp, email_templates;

type
  TNotificationType = (ntInfo, ntSuccess, ntWarning, ntError, ntAlert);
  
  TEmailNotifier = class
  private
    FSMTPClient: TSMTPClient;
    FFromEmail: string;
    FFromName: string;
    FCompanyName: string;
    FTemplateDir: string;
    FQueue: TStringList;
    FQueueEnabled: Boolean;
    
    function TypeToStyle(AType: TNotificationType): string;
    function TypeToIcon(AType: TNotificationType): string;
    function BuildNotificationHTML(const ATitle, AMessage: string; 
      AType: TNotificationType; const AActionLink, AActionText: string): string;
    
  public
    constructor Create(const AConfig: TSMTPConfig; 
      const AFromEmail, AFromName, ACompanyName: string);
    destructor Destroy; override;
    
    // ส่ง notifications ประเภทต่างๆ
    function SendNotification(const AToEmail, ASubject, ATitle, AMessage: string;
      AType: TNotificationType = ntInfo;
      const AActionLink: string = ''; const AActionText: string = ''): Boolean;
      
    function SendWelcome(const AToEmail, AToName, AActivationLink: string): Boolean;
    
    function SendOrderConfirm(const AToEmail, AToName, AOrderNo: string;
      ATotal: Double): Boolean;
      
    function SendPasswordReset(const AToEmail, AToName, AResetLink: string): Boolean;
    
    function SendBulk(const ARecipients: TStringList; const ASubject: string;
      AMessage: TEmailMessage): Integer;
      
    function SendWithAttachment(const AToEmail, ASubject, ABody: string;
      const AAttachmentFile: string): Boolean;
      
    // Queue management
    procedure QueueEmail(const AToEmail, ASubject, ABody: string);
    function ProcessQueue: Integer;
    
    property FromEmail: string read FFromEmail write FFromEmail;
    property FromName: string read FFromName write FFromName;
    property CompanyName: string read FCompanyName write FCompanyName;
    property QueueEnabled: Boolean read FQueueEnabled write FQueueEnabled;
  end;

implementation

{ TEmailNotifier }

constructor TEmailNotifier.Create(const AConfig: TSMTPConfig;
  const AFromEmail, AFromName, ACompanyName: string);
begin
  inherited Create;
  FSMTPClient := TSMTPClient.Create(AConfig);
  FFromEmail := AFromEmail;
  FFromName := AFromName;
  FCompanyName := ACompanyName;
  FQueue := TStringList.Create;
  FQueueEnabled := False;
end;

destructor TEmailNotifier.Destroy;
begin
  FQueue.Free;
  FSMTPClient.Free;
  inherited Destroy;
end;

function TEmailNotifier.TypeToStyle(AType: TNotificationType): string;
begin
  case AType of
    ntInfo:    Result := 'background:#3498db;';
    ntSuccess: Result := 'background:#27ae60;';
    ntWarning: Result := 'background:#e67e22;';
    ntError:   Result := 'background:#e74c3c;';
    ntAlert:   Result := 'background:#8e44ad;';
    else       Result := 'background:#95a5a6;';
  end;
end;

function TEmailNotifier.TypeToIcon(AType: TNotificationType): string;
begin
  case AType of
    ntInfo:    Result := 'ℹ️';
    ntSuccess: Result := '✅';
    ntWarning: Result := '⚠️';
    ntError:   Result := '❌';
    ntAlert:   Result := '🔔';
    else       Result := '📢';
  end;
end;

function TEmailNotifier.BuildNotificationHTML(const ATitle, AMessage: string;
  AType: TNotificationType; const AActionLink, AActionText: string): string;
var
  Style: string;
  Icon: string;
begin
  Style := TypeToStyle(AType);
  Icon := TypeToIcon(AType);
  
  Result :=
    '<!DOCTYPE html>' +
    '<html lang="th"><head><meta charset="UTF-8">' +
    '<style>' +
    'body{font-family:sans-serif;background:#f4f4f4;margin:0;padding:20px;}' +
    '.card{max-width:500px;margin:auto;background:white;border-radius:8px;overflow:hidden;}' +
    '.banner{' + Style + 'color:white;padding:20px;text-align:center;font-size:18px;font-weight:bold;}' +
    '.body{padding:25px;color:#333;}' +
    '.footer{background:#f8f9fa;padding:15px;text-align:center;color:#666;font-size:12px;}' +
    '.btn{display:inline-block;padding:10px 20px;' + Style + 'color:white;' +
    'text-decoration:none;border-radius:4px;margin:10px 0;}' +
    '</style></head><body>' +
    '<div class="card">' +
    '<div class="banner">' + Icon + ' ' + ATitle + '</div>' +
    '<div class="body"><p>' + AMessage + '</p>';
    
  if (AActionLink <> '') and (AActionText <> '') then
    Result := Result + 
      '<p><a class="btn" href="' + AActionLink + '">' + AActionText + '</a></p>';
      
  Result := Result +
    '</div>' +
    '<div class="footer">' + FCompanyName + ' - ' + 
    FormatDateTime('dd/mm/yyyy hh:nn', Now) + '</div>' +
    '</div></body></html>';
end;

function TEmailNotifier.SendNotification(const AToEmail, ASubject, ATitle, 
  AMessage: string; AType: TNotificationType; const AActionLink, AActionText: string): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := FFromEmail;
    Msg.FromName := FFromName;
    Msg.AddTo(AToEmail);
    Msg.Subject := ASubject;
    Msg.Body := ATitle + #13#10#13#10 + AMessage;
    Msg.HTMLBody := BuildNotificationHTML(ATitle, AMessage, AType, AActionLink, AActionText);
    
    Result := FSMTPClient.SendMessage(Msg);
    
    if not Result then
      WriteLn('ส่ง notification ล้มเหลว: ', FSMTPClient.LastError);
  finally
    Msg.Free;
  end;
end;

function TEmailNotifier.SendWelcome(const AToEmail, AToName, AActivationLink: string): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := FFromEmail;
    Msg.FromName := FFromName;
    Msg.AddTo(AToEmail);
    Msg.Subject := 'ยินดีต้อนรับสู่ ' + FCompanyName + '!';
    Msg.Body := 'สวัสดี ' + AToName + ', กรุณายืนยันบัญชีที่: ' + AActivationLink;
    Msg.HTMLBody := WelcomeEmailHTML(AToName, AActivationLink);
    
    Result := FSMTPClient.SendMessage(Msg);
  finally
    Msg.Free;
  end;
end;

function TEmailNotifier.SendOrderConfirm(const AToEmail, AToName, AOrderNo: string;
  ATotal: Double): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := FFromEmail;
    Msg.FromName := FFromName;
    Msg.AddTo(AToEmail);
    Msg.Subject := 'ยืนยันคำสั่งซื้อ #' + AOrderNo;
    Msg.Body := 'ขอบคุณ ' + AToName + ' สำหรับคำสั่งซื้อ #' + AOrderNo;
    Msg.HTMLBody := OrderConfirmHTML(AOrderNo, AToName, ATotal);
    
    Result := FSMTPClient.SendMessage(Msg);
  finally
    Msg.Free;
  end;
end;

function TEmailNotifier.SendPasswordReset(const AToEmail, AToName, AResetLink: string): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := FFromEmail;
    Msg.FromName := FFromName;
    Msg.AddTo(AToEmail);
    Msg.Subject := 'รีเซ็ตรหัสผ่าน ' + FCompanyName;
    Msg.Body := 'คลิก link นี้เพื่อรีเซ็ตรหัสผ่าน: ' + AResetLink;
    Msg.HTMLBody := PasswordResetHTML(AToName, AResetLink);
    
    Result := FSMTPClient.SendMessage(Msg);
  finally
    Msg.Free;
  end;
end;

function TEmailNotifier.SendBulk(const ARecipients: TStringList; 
  const ASubject: string; AMessage: TEmailMessage): Integer;
var
  i: Integer;
  Msg: TEmailMessage;
begin
  Result := 0;
  
  for i := 0 to ARecipients.Count - 1 do
  begin
    Msg := TEmailMessage.Create;
    try
      Msg.From := FFromEmail;
      Msg.FromName := FFromName;
      Msg.AddTo(ARecipients[i]);
      Msg.Subject := ASubject;
      Msg.Body := AMessage.Body;
      Msg.HTMLBody := AMessage.HTMLBody;
      
      if FSMTPClient.SendMessage(Msg) then
      begin
        Inc(Result);
        WriteLn('ส่งสำเร็จ: ', ARecipients[i]);
      end
      else
        WriteLn('ส่งล้มเหลว: ', ARecipients[i], ' - ', FSMTPClient.LastError);
        
      // หน่วงเวลาเล็กน้อยระหว่าง emails
      Sleep(500);
      
    finally
      Msg.Free;
    end;
  end;
  
  WriteLn(Format('ส่งสำเร็จ %d/%d', [Result, ARecipients.Count]));
end;

function TEmailNotifier.SendWithAttachment(const AToEmail, ASubject, ABody: string;
  const AAttachmentFile: string): Boolean;
var
  Msg: TEmailMessage;
begin
  Msg := TEmailMessage.Create;
  try
    Msg.From := FFromEmail;
    Msg.FromName := FFromName;
    Msg.AddTo(AToEmail);
    Msg.Subject := ASubject;
    Msg.Body := ABody;
    
    if FileExists(AAttachmentFile) then
      Msg.AddAttachment(AAttachmentFile);
      
    Result := FSMTPClient.SendMessage(Msg);
  finally
    Msg.Free;
  end;
end;

procedure TEmailNotifier.QueueEmail(const AToEmail, ASubject, ABody: string);
begin
  FQueue.Add(AToEmail + '|' + ASubject + '|' + ABody);
end;

function TEmailNotifier.ProcessQueue: Integer;
var
  i: Integer;
  Parts: TStringList;
begin
  Result := 0;
  Parts := TStringList.Create;
  try
    for i := FQueue.Count - 1 downto 0 do
    begin
      Parts.Delimiter := '|';
      Parts.DelimitedText := FQueue[i];
      
      if Parts.Count >= 3 then
      begin
        if FSMTPClient.SendSimple(FFromEmail, Parts[0], Parts[1], Parts[2]) then
        begin
          Inc(Result);
          FQueue.Delete(i);
        end;
      end;
    end;
  finally
    Parts.Free;
  end;
end;

end.
```

---

## 68.5 ตัวอย่างโปรแกรมสมบูรณ์

```pascal
program email_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  email_smtp, email_templates, notification_system;

procedure DemoSendEmails;
var
  Config: TSMTPConfig;
  Notifier: TEmailNotifier;
  Recipients: TStringList;
begin
  // ตั้งค่า SMTP (Gmail)
  Config := GmailConfig('your-email@gmail.com', 'your-app-password');
  
  Notifier := TEmailNotifier.Create(
    Config,
    'your-email@gmail.com',
    'MyApp Team',
    'MyApp'
  );
  
  try
    // 1. ส่ง Welcome Email
    WriteLn('1. ส่ง Welcome Email...');
    Notifier.SendWelcome(
      'user@example.com',
      'สมชาย รักเรียน',
      'https://myapp.com/activate?token=abc123'
    );
    
    // 2. ส่ง Order Confirmation
    WriteLn('2. ส่ง Order Confirmation...');
    Notifier.SendOrderConfirm(
      'customer@example.com',
      'สมหญิง ใจดี',
      'ORD-2024-00001',
      1500.00
    );
    
    // 3. ส่ง Notification
    WriteLn('3. ส่ง Alert Notification...');
    Notifier.SendNotification(
      'admin@example.com',
      'แจ้งเตือน: Server Load สูง',
      'CPU Usage สูงกว่า 90%',
      'Server main-01 มี CPU usage สูงถึง 95% กรุณาตรวจสอบด่วน',
      ntAlert,
      'https://admin.myapp.com/servers',
      'ดูรายละเอียด'
    );
    
    // 4. Bulk Email
    WriteLn('4. ส่ง Bulk Email...');
    Recipients := TStringList.Create;
    try
      Recipients.Add('user1@example.com');
      Recipients.Add('user2@example.com');
      Recipients.Add('user3@example.com');
      
      var Msg := TEmailMessage.Create;
      try
        Msg.Subject := 'Newsletter ประจำเดือน';
        Msg.Body := 'ข่าวสารประจำเดือน...';
        Msg.HTMLBody := NotificationHTML(
          'Newsletter ประจำเดือน',
          'สวัสดี! นี่คือข่าวสารล่าสุดจาก MyApp...',
          'https://myapp.com/newsletter',
          'อ่านเพิ่มเติม'
        );
        
        var Sent := Notifier.SendBulk(Recipients, Msg.Subject, Msg);
        WriteLn('ส่งสำเร็จ: ', Sent, ' emails');
      finally
        Msg.Free;
      end;
    finally
      Recipients.Free;
    end;
    
    // 5. Email with attachment
    WriteLn('5. ส่ง Email with Attachment...');
    Notifier.SendWithAttachment(
      'boss@example.com',
      'รายงานยอดขายประจำเดือน',
      'รายงานยอดขายแนบมาด้วย',
      'sales_report.xlsx'
    );
    
    WriteLn('ส่ง Email ทั้งหมดสำเร็จ');
    
  finally
    Notifier.Free;
  end;
end;

begin
  WriteLn('=== Email System Demo ===');
  DemoSendEmails;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **SMTP Client** - ส่ง email ผ่าน SMTP
2. **MIME Message** - สร้าง email ที่มี HTML และ attachments
3. **HTML Templates** - สร้าง email template สวยงาม
4. **Email Notifier** - ระบบแจ้งเตือนสมบูรณ์
5. **Bulk Email** - ส่ง email จำนวนมาก

ข้อควรระวัง:
- ต้องใช้ App Password สำหรับ Gmail (ไม่ใช่รหัสผ่าน account จริง)
- หน่วงเวลาระหว่างการส่ง bulk emails เพื่อไม่โดน spam filter
- เก็บรหัสผ่านใน environment variables ไม่ใช่ hardcode ในโค้ด
