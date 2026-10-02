# Part 56 - Security Programming ใน Pascal/Lazarus

## บทนำ

ความปลอดภัยในโปรแกรมมีความสำคัญอย่างมาก การเขียนโค้ดที่ไม่ปลอดภัยอาจนำไปสู่การโจมตีต่างๆ เช่น SQL Injection, XSS, Buffer Overflow และการรั่วไหลของข้อมูล

---

## 1. Input Validation

```pascal
unit InputValidation;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, RegExpr;

type
  TValidationResult = record
    IsValid: Boolean;
    ErrorMessage: string;
  end;

  TInputValidator = class
  public
    { String Validators }
    class function ValidateEmail(const AEmail: string): TValidationResult;
    class function ValidatePhone(const APhone: string): TValidationResult;
    class function ValidateUsername(const AUsername: string): TValidationResult;
    class function ValidatePassword(const APassword: string): TValidationResult;
    class function ValidateURL(const AURL: string): TValidationResult;
    class function ValidateSQLSafe(const AInput: string): TValidationResult;
    
    { Numeric Validators }
    class function ValidateInt(const AValue: string; AMin, AMax: Int64): TValidationResult;
    class function ValidateFloat(const AValue: string; AMin, AMax: Double): TValidationResult;
    
    { File Validators }
    class function ValidateFilename(const AFilename: string): TValidationResult;
    class function ValidateFileExtension(const AFilename: string; 
                                          const AAllowed: array of string): TValidationResult;
    class function ValidateFileSize(ASize: Int64; AMaxSize: Int64): TValidationResult;
    
    { Sanitizers }
    class function SanitizeHTML(const AInput: string): string;
    class function SanitizeSQL(const AInput: string): string;
    class function SanitizeFilename(const AFilename: string): string;
    class function StripTags(const AHTML: string): string;
    class function EscapeJSON(const AInput: string): string;
  end;

implementation

class function TInputValidator.ValidateEmail(const AEmail: string): TValidationResult;
var
  Regex: TRegExpr;
begin
  Result.ErrorMessage := '';
  
  if Trim(AEmail) = '' then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'อีเมลต้องไม่ว่าง';
    Exit;
  end;
  
  if Length(AEmail) > 254 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'อีเมลยาวเกินไป (สูงสุด 254 ตัวอักษร)';
    Exit;
  end;
  
  Regex := TRegExpr.Create('^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$');
  try
    Result.IsValid := Regex.Exec(AEmail);
    if not Result.IsValid then
      Result.ErrorMessage := 'รูปแบบอีเมลไม่ถูกต้อง';
  finally
    Regex.Free;
  end;
end;

class function TInputValidator.ValidatePhone(const APhone: string): TValidationResult;
var
  CleanPhone: string;
  I: Integer;
begin
  CleanPhone := '';
  for I := 1 to Length(APhone) do
    if APhone[I] in ['0'..'9', '+', '-', ' ', '(', ')'] then
      CleanPhone := CleanPhone + APhone[I];
  
  CleanPhone := StringReplace(CleanPhone, ' ', '', [rfReplaceAll]);
  CleanPhone := StringReplace(CleanPhone, '-', '', [rfReplaceAll]);
  CleanPhone := StringReplace(CleanPhone, '(', '', [rfReplaceAll]);
  CleanPhone := StringReplace(CleanPhone, ')', '', [rfReplaceAll]);
  
  if Length(CleanPhone) < 9 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'เบอร์โทรต้องมีอย่างน้อย 9 หลัก';
  end
  else if Length(CleanPhone) > 15 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'เบอร์โทรยาวเกินไป (สูงสุด 15 หลัก)';
  end
  else
  begin
    Result.IsValid := True;
    Result.ErrorMessage := '';
  end;
end;

class function TInputValidator.ValidateUsername(const AUsername: string): TValidationResult;
var
  Regex: TRegExpr;
begin
  if Length(AUsername) < 3 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร';
    Exit;
  end;
  
  if Length(AUsername) > 30 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ชื่อผู้ใช้ยาวเกินไป (สูงสุด 30 ตัวอักษร)';
    Exit;
  end;
  
  Regex := TRegExpr.Create('^[a-zA-Z0-9_]+$');
  try
    Result.IsValid := Regex.Exec(AUsername);
    if not Result.IsValid then
      Result.ErrorMessage := 'ชื่อผู้ใช้ใช้ได้เฉพาะ a-z, A-Z, 0-9 และ _ เท่านั้น';
  finally
    Regex.Free;
  end;
end;

class function TInputValidator.ValidatePassword(const APassword: string): TValidationResult;
var
  HasUpper, HasLower, HasDigit, HasSpecial: Boolean;
  C: Char;
begin
  Result.IsValid := False;
  
  if Length(APassword) < 8 then
  begin
    Result.ErrorMessage := 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    Exit;
  end;
  
  if Length(APassword) > 128 then
  begin
    Result.ErrorMessage := 'รหัสผ่านยาวเกินไป (สูงสุด 128 ตัวอักษร)';
    Exit;
  end;
  
  HasUpper := False; HasLower := False;
  HasDigit := False; HasSpecial := False;
  
  for C in APassword do
  begin
    if C in ['A'..'Z'] then HasUpper := True
    else if C in ['a'..'z'] then HasLower := True
    else if C in ['0'..'9'] then HasDigit := True
    else HasSpecial := True;
  end;
  
  if not HasUpper then
  begin
    Result.ErrorMessage := 'รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัว';
    Exit;
  end;
  
  if not HasLower then
  begin
    Result.ErrorMessage := 'รหัสผ่านต้องมีตัวอักษรพิมพ์เล็กอย่างน้อย 1 ตัว';
    Exit;
  end;
  
  if not HasDigit then
  begin
    Result.ErrorMessage := 'รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว';
    Exit;
  end;
  
  if not HasSpecial then
  begin
    Result.ErrorMessage := 'รหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว (!@#$%^&*)';
    Exit;
  end;
  
  Result.IsValid := True;
  Result.ErrorMessage := '';
end;

class function TInputValidator.ValidateURL(const AURL: string): TValidationResult;
var
  Regex: TRegExpr;
begin
  Regex := TRegExpr.Create(
    '^https?://[a-zA-Z0-9\-]+(\.[a-zA-Z0-9\-]+)*(:[0-9]+)?(/[^\s]*)?$');
  try
    Result.IsValid := Regex.Exec(AURL);
    if not Result.IsValid then
      Result.ErrorMessage := 'รูปแบบ URL ไม่ถูกต้อง';
  finally
    Regex.Free;
  end;
end;

class function TInputValidator.ValidateSQLSafe(const AInput: string): TValidationResult;
const
  SQL_KEYWORDS: array[0..14] of string = (
    'DROP', 'DELETE', 'INSERT', 'UPDATE', 'SELECT', 
    'UNION', 'OR 1=1', 'OR 1 = 1', '; --', '/*',
    'EXEC', 'EXECUTE', 'CAST(', 'CONVERT(', 'WAITFOR'
  );
var
  UpperInput: string;
  Keyword: string;
begin
  UpperInput := UpperCase(AInput);
  
  for Keyword in SQL_KEYWORDS do
    if Pos(Keyword, UpperInput) > 0 then
    begin
      Result.IsValid := False;
      Result.ErrorMessage := 'ตรวจพบ SQL Injection Attempt';
      Exit;
    end;
  
  { ตรวจสอบ Single Quote ที่ไม่ได้ Escape }
  if (Pos('''', AInput) > 0) and (Pos('''''', AInput) = 0) then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ตรวจพบ SQL Injection: Single Quote ที่ไม่ปลอดภัย';
    Exit;
  end;
  
  Result.IsValid := True;
  Result.ErrorMessage := '';
end;

class function TInputValidator.ValidateInt(const AValue: string; AMin, AMax: Int64): TValidationResult;
var
  N: Int64;
  Code: Integer;
begin
  Val(AValue, N, Code);
  
  if Code <> 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ค่าต้องเป็นตัวเลข';
    Exit;
  end;
  
  if N < AMin then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := Format('ค่าต้องไม่น้อยกว่า %d', [AMin]);
    Exit;
  end;
  
  if N > AMax then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := Format('ค่าต้องไม่มากกว่า %d', [AMax]);
    Exit;
  end;
  
  Result.IsValid := True;
  Result.ErrorMessage := '';
end;

class function TInputValidator.ValidateFloat(const AValue: string; AMin, AMax: Double): TValidationResult;
var
  N: Double;
  Code: Integer;
begin
  Val(AValue, N, Code);
  
  if Code <> 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ค่าต้องเป็นตัวเลข';
    Exit;
  end;
  
  if N < AMin then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := Format('ค่าต้องไม่น้อยกว่า %.2f', [AMin]);
    Exit;
  end;
  
  if N > AMax then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := Format('ค่าต้องไม่มากกว่า %.2f', [AMax]);
    Exit;
  end;
  
  Result.IsValid := True;
  Result.ErrorMessage := '';
end;

class function TInputValidator.ValidateFilename(const AFilename: string): TValidationResult;
const
  FORBIDDEN_CHARS = '<>:"/\|?*';
var
  C: Char;
  Name: string;
begin
  Name := ExtractFileName(AFilename);
  
  if Name = '' then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ชื่อไฟล์ต้องไม่ว่าง';
    Exit;
  end;
  
  for C in Name do
    if C in [#0..#31] then
    begin
      Result.IsValid := False;
      Result.ErrorMessage := 'ชื่อไฟล์มีอักขระที่ไม่อนุญาต';
      Exit;
    end;
  
  if (Pos('..', AFilename) > 0) then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := 'ชื่อไฟล์ไม่อนุญาตให้ใช้ ".."';
    Exit;
  end;
  
  Result.IsValid := True;
  Result.ErrorMessage := '';
end;

class function TInputValidator.ValidateFileExtension(const AFilename: string;
  const AAllowed: array of string): TValidationResult;
var
  Ext: string;
  AllowedExt: string;
begin
  Ext := LowerCase(ExtractFileExt(AFilename));
  
  for AllowedExt in AAllowed do
    if LowerCase(AllowedExt) = Ext then
    begin
      Result.IsValid := True;
      Result.ErrorMessage := '';
      Exit;
    end;
  
  Result.IsValid := False;
  Result.ErrorMessage := Format('ประเภทไฟล์ %s ไม่อนุญาต', [Ext]);
end;

class function TInputValidator.ValidateFileSize(ASize: Int64; AMaxSize: Int64): TValidationResult;
begin
  if ASize > AMaxSize then
  begin
    Result.IsValid := False;
    Result.ErrorMessage := Format('ไฟล์ขนาดใหญ่เกินไป (สูงสุด %d MB)', 
      [AMaxSize div (1024 * 1024)]);
  end
  else
  begin
    Result.IsValid := True;
    Result.ErrorMessage := '';
  end;
end;

class function TInputValidator.SanitizeHTML(const AInput: string): string;
begin
  Result := AInput;
  Result := StringReplace(Result, '&', '&amp;', [rfReplaceAll]);
  Result := StringReplace(Result, '<', '&lt;', [rfReplaceAll]);
  Result := StringReplace(Result, '>', '&gt;', [rfReplaceAll]);
  Result := StringReplace(Result, '"', '&quot;', [rfReplaceAll]);
  Result := StringReplace(Result, '''', '&#x27;', [rfReplaceAll]);
  Result := StringReplace(Result, '/', '&#x2F;', [rfReplaceAll]);
end;

class function TInputValidator.SanitizeSQL(const AInput: string): string;
begin
  { Escape Single Quote }
  Result := StringReplace(AInput, '''', '''''', [rfReplaceAll]);
  Result := StringReplace(Result, '\', '\\', [rfReplaceAll]);
  Result := StringReplace(Result, #0, '\0', [rfReplaceAll]);
  Result := StringReplace(Result, #26, '\Z', [rfReplaceAll]);
end;

class function TInputValidator.SanitizeFilename(const AFilename: string): string;
const
  FORBIDDEN_CHARS = '<>:"/\|?*';
var
  C: Char;
begin
  Result := '';
  for C in AFilename do
  begin
    if C in [#0..#31] then Continue;
    if Pos(C, FORBIDDEN_CHARS) > 0 then
      Result := Result + '_'
    else
      Result := Result + C;
  end;
  
  { ลบ .. เพื่อป้องกัน Path Traversal }
  while Pos('..', Result) > 0 do
    Result := StringReplace(Result, '..', '_', [rfReplaceAll]);
end;

class function TInputValidator.StripTags(const AHTML: string): string;
var
  InTag: Boolean;
  I: Integer;
begin
  Result := '';
  InTag := False;
  for I := 1 to Length(AHTML) do
  begin
    if AHTML[I] = '<' then InTag := True
    else if AHTML[I] = '>' then InTag := False
    else if not InTag then
      Result := Result + AHTML[I];
  end;
end;

class function TInputValidator.EscapeJSON(const AInput: string): string;
var
  I: Integer;
  C: Char;
begin
  Result := '';
  for I := 1 to Length(AInput) do
  begin
    C := AInput[I];
    case C of
      '"':  Result := Result + '\"';
      '\':  Result := Result + '\\';
      '/':  Result := Result + '\/';
      #8:   Result := Result + '\b';
      #9:   Result := Result + '\t';
      #10:  Result := Result + '\n';
      #12:  Result := Result + '\f';
      #13:  Result := Result + '\r';
    else
      if Ord(C) < 32 then
        Result := Result + Format('\u%4.4x', [Ord(C)])
      else
        Result := Result + C;
    end;
  end;
end;

end.
```

---

## 2. Password Hashing

```pascal
unit PasswordHashing;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sha256, md5;

type
  { Password Hasher - BCrypt-like using SHA-256 + Salt }
  TPasswordHasher = class
  private
    class function GenerateSalt(ALength: Integer = 32): string;
    class function SHA256Hex(const AData: string): string;
    class function PBKDF2(const APassword, ASalt: string; 
                           AIterations, AKeyLength: Integer): string;
  public
    class function HashPassword(const APassword: string): string;
    class function VerifyPassword(const APassword, AHash: string): Boolean;
    class function NeedsRehash(const AHash: string): Boolean;
  end;

  { Rate Limiter สำหรับ Login }
  TLoginRateLimiter = class
  private
    FAttempts: TStringList;
    FMaxAttempts: Integer;
    FWindowSeconds: Integer;
    FLockoutSeconds: Integer;
    
    procedure CleanOldAttempts;
    function GetKey(const AIP, AUsername: string): string;
  public
    constructor Create;
    destructor Destroy; override;
    
    function IsAllowed(const AIP, AUsername: string): Boolean;
    procedure RecordAttempt(const AIP, AUsername: string; ASuccess: Boolean);
    function GetRemainingAttempts(const AIP, AUsername: string): Integer;
    function GetLockoutRemainingSeconds(const AIP, AUsername: string): Integer;
    procedure Reset(const AIP, AUsername: string);
    
    property MaxAttempts: Integer read FMaxAttempts write FMaxAttempts;
    property WindowSeconds: Integer read FWindowSeconds write FWindowSeconds;
    property LockoutSeconds: Integer read FLockoutSeconds write FLockoutSeconds;
  end;

implementation

class function TPasswordHasher.GenerateSalt(ALength: Integer): string;
const
  CHARS = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789./';
var
  I: Integer;
begin
  Result := '';
  for I := 1 to ALength do
    Result := Result + CHARS[Random(Length(CHARS)) + 1];
end;

class function TPasswordHasher.SHA256Hex(const AData: string): string;
var
  Hash: TSHA256Digest;
  I: Integer;
begin
  Hash := SHA256String(AData);
  Result := '';
  for I := 0 to 31 do
    Result := Result + IntToHex(Hash[I], 2);
  Result := LowerCase(Result);
end;

class function TPasswordHasher.PBKDF2(const APassword, ASalt: string;
  AIterations, AKeyLength: Integer): string;
var
  I: Integer;
  U: string;
  T: string;
begin
  { Simple PBKDF2 simulation using SHA256 }
  T := '';
  U := APassword + ASalt;
  
  for I := 1 to AIterations do
    U := SHA256Hex(U + IntToStr(I));
  
  Result := Copy(U, 1, AKeyLength * 2);
end;

class function TPasswordHasher.HashPassword(const APassword: string): string;
var
  Salt: string;
  Hash: string;
const
  ITERATIONS = 10000;
  KEY_LENGTH = 32;
  VERSION = 1;
begin
  Salt := GenerateSalt(32);
  Hash := PBKDF2(APassword, Salt, ITERATIONS, KEY_LENGTH);
  
  { รูปแบบ: $v1$iterations$salt$hash }
  Result := Format('$v%d$%d$%s$%s', [VERSION, ITERATIONS, Salt, Hash]);
end;

class function TPasswordHasher.VerifyPassword(const APassword, AHash: string): Boolean;
var
  Parts: TStringArray;
  Version, Iterations: Integer;
  Salt, StoredHash, ComputedHash: string;
begin
  Result := False;
  
  if Copy(AHash, 1, 1) <> '$' then Exit;
  
  Parts := AHash.Split(['$']);
  { ['', 'v1', '10000', 'salt', 'hash'] }
  if Length(Parts) < 5 then Exit;
  
  try
    Version := StrToInt(Copy(Parts[1], 2, MaxInt));
    Iterations := StrToInt(Parts[2]);
    Salt := Parts[3];
    StoredHash := Parts[4];
    
    if Version <> 1 then Exit;
    
    ComputedHash := PBKDF2(APassword, Salt, Iterations, 32);
    Result := ComputedHash = StoredHash;
  except
    Result := False;
  end;
end;

class function TPasswordHasher.NeedsRehash(const AHash: string): Boolean;
var
  Parts: TStringArray;
  Iterations: Integer;
const
  MIN_ITERATIONS = 10000;
begin
  Result := True;
  
  Parts := AHash.Split(['$']);
  if Length(Parts) < 5 then Exit;
  
  try
    Iterations := StrToInt(Parts[2]);
    Result := Iterations < MIN_ITERATIONS;
  except
    Result := True;
  end;
end;

{ TLoginRateLimiter }
constructor TLoginRateLimiter.Create;
begin
  FAttempts := TStringList.Create;
  FMaxAttempts := 5;
  FWindowSeconds := 300;  { 5 นาที }
  FLockoutSeconds := 900; { 15 นาที }
end;

destructor TLoginRateLimiter.Destroy;
begin
  FAttempts.Free;
  inherited;
end;

function TLoginRateLimiter.GetKey(const AIP, AUsername: string): string;
begin
  Result := AIP + ':' + LowerCase(AUsername);
end;

procedure TLoginRateLimiter.CleanOldAttempts;
var
  I: Integer;
  Cutoff: TDateTime;
  AttemptTime: TDateTime;
begin
  Cutoff := Now - (FWindowSeconds / 86400.0);
  for I := FAttempts.Count - 1 downto 0 do
  begin
    AttemptTime := StrToDateTime(FAttempts.ValueFromIndex[I]);
    if AttemptTime < Cutoff then
      FAttempts.Delete(I);
  end;
end;

function TLoginRateLimiter.IsAllowed(const AIP, AUsername: string): Boolean;
var
  Key: string;
  Count: Integer;
  I: Integer;
  Cutoff: TDateTime;
  AttemptTime: TDateTime;
  Parts: TStringArray;
begin
  Key := GetKey(AIP, AUsername);
  Count := 0;
  Cutoff := Now - (FWindowSeconds / 86400.0);
  
  for I := 0 to FAttempts.Count - 1 do
  begin
    if FAttempts.Names[I] = Key then
    begin
      Parts := FAttempts.ValueFromIndex[I].Split(['|']);
      if Length(Parts) >= 2 then
      begin
        AttemptTime := StrToDateTimeDef(Parts[0], 0);
        if (AttemptTime >= Cutoff) and (Parts[1] = 'fail') then
          Inc(Count);
      end;
    end;
  end;
  
  Result := Count < FMaxAttempts;
end;

procedure TLoginRateLimiter.RecordAttempt(const AIP, AUsername: string; ASuccess: Boolean);
var
  Key: string;
  Status: string;
begin
  Key := GetKey(AIP, AUsername);
  if ASuccess then
    Status := 'success'
  else
    Status := 'fail';
  
  FAttempts.Add(Key + '=' + 
    FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + '|' + Status);
  
  { ทำความสะอาด }
  if FAttempts.Count > 10000 then
    CleanOldAttempts;
end;

function TLoginRateLimiter.GetRemainingAttempts(const AIP, AUsername: string): Integer;
var
  Key: string;
  Count: Integer;
  I: Integer;
  Cutoff: TDateTime;
  AttemptTime: TDateTime;
  Parts: TStringArray;
begin
  Key := GetKey(AIP, AUsername);
  Count := 0;
  Cutoff := Now - (FWindowSeconds / 86400.0);
  
  for I := 0 to FAttempts.Count - 1 do
    if FAttempts.Names[I] = Key then
    begin
      Parts := FAttempts.ValueFromIndex[I].Split(['|']);
      if (Length(Parts) >= 2) then
      begin
        AttemptTime := StrToDateTimeDef(Parts[0], 0);
        if (AttemptTime >= Cutoff) and (Parts[1] = 'fail') then
          Inc(Count);
      end;
    end;
  
  Result := Max(0, FMaxAttempts - Count);
end;

function TLoginRateLimiter.GetLockoutRemainingSeconds(const AIP, AUsername: string): Integer;
begin
  if IsAllowed(AIP, AUsername) then
    Result := 0
  else
    Result := FLockoutSeconds;
end;

procedure TLoginRateLimiter.Reset(const AIP, AUsername: string);
var
  Key: string;
  I: Integer;
begin
  Key := GetKey(AIP, AUsername);
  for I := FAttempts.Count - 1 downto 0 do
    if FAttempts.Names[I] = Key then
      FAttempts.Delete(I);
end;

initialization
  Randomize;

end.
```

---

## 3. Secure Database Access

```pascal
unit SecureDatabase;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb, sqlite3conn;

type
  { Parameterized Query Helper }
  TSecureQuery = class
  private
    FConnection: TSQLConnection;
    FQuery: TSQLQuery;
    FTransaction: TSQLTransaction;
  public
    constructor Create(AConnection: TSQLConnection);
    destructor Destroy; override;
    
    { Prepared Statement Methods }
    function PrepareAndExecute(const ASQL: string; 
                                const AParams: array of Variant): Boolean;
    function PrepareAndOpen(const ASQL: string;
                             const AParams: array of Variant): Boolean;
    
    { Safe Query Methods }
    function SelectOne(const ATable, AWhere: string; 
                        const AWhereParams: array of Variant): Boolean;
    function SelectAll(const ATable: string; 
                        const AConditions: array of string;
                        const AParams: array of Variant;
                        APage: Integer = 0; APageSize: Integer = 0): Boolean;
    function Insert(const ATable: string; 
                    const AColumns: array of string;
                    const AValues: array of Variant): Integer;
    function Update_(const ATable: string;
                     const ASetColumns: array of string;
                     const ASetValues: array of Variant;
                     const AWhere: string;
                     const AWhereParams: array of Variant): Integer;
    function Delete_(const ATable: string;
                     const AWhere: string;
                     const AWhereParams: array of Variant): Integer;
    
    property Query: TSQLQuery read FQuery;
  end;

implementation

constructor TSecureQuery.Create(AConnection: TSQLConnection);
begin
  FConnection := AConnection;
  FTransaction := TSQLTransaction.Create(nil);
  FTransaction.DataBase := AConnection;
  FQuery := TSQLQuery.Create(nil);
  FQuery.DataBase := AConnection;
  FQuery.Transaction := FTransaction;
end;

destructor TSecureQuery.Destroy;
begin
  FQuery.Free;
  FTransaction.Free;
  inherited;
end;

function TSecureQuery.PrepareAndExecute(const ASQL: string;
  const AParams: array of Variant): Boolean;
var
  I: Integer;
begin
  FTransaction.StartTransaction;
  try
    FQuery.SQL.Text := ASQL;
    FQuery.Prepare;
    
    for I := 0 to High(AParams) do
      FQuery.Params[I].Value := AParams[I];
    
    FQuery.ExecSQL;
    FTransaction.Commit;
    Result := True;
  except
    FTransaction.Rollback;
    raise;
  end;
end;

function TSecureQuery.PrepareAndOpen(const ASQL: string;
  const AParams: array of Variant): Boolean;
var
  I: Integer;
begin
  FQuery.SQL.Text := ASQL;
  FQuery.Prepare;
  
  for I := 0 to High(AParams) do
    FQuery.Params[I].Value := AParams[I];
  
  FQuery.Open;
  Result := not FQuery.EOF;
end;

function TSecureQuery.SelectOne(const ATable, AWhere: string;
  const AWhereParams: array of Variant): Boolean;
var
  SQL: string;
begin
  { ตรวจสอบ Table Name (whitelist) }
  SQL := Format('SELECT * FROM %s WHERE %s LIMIT 1', 
    [QuotedStr(ATable), AWhere]);
  Result := PrepareAndOpen(SQL, AWhereParams);
end;

function TSecureQuery.SelectAll(const ATable: string;
  const AConditions: array of string;
  const AParams: array of Variant;
  APage: Integer; APageSize: Integer): Boolean;
var
  SQL: string;
  WhereClause: string;
  I: Integer;
begin
  WhereClause := '';
  for I := 0 to High(AConditions) do
  begin
    if I > 0 then WhereClause := WhereClause + ' AND ';
    WhereClause := WhereClause + AConditions[I];
  end;
  
  if WhereClause <> '' then
    SQL := Format('SELECT * FROM %s WHERE %s', [ATable, WhereClause])
  else
    SQL := 'SELECT * FROM ' + ATable;
  
  if (APage > 0) and (APageSize > 0) then
    SQL := SQL + Format(' LIMIT %d OFFSET %d',
      [APageSize, (APage - 1) * APageSize]);
  
  Result := PrepareAndOpen(SQL, AParams);
end;

function TSecureQuery.Insert(const ATable: string;
  const AColumns: array of string;
  const AValues: array of Variant): Integer;
var
  SQL, ColList, ParamList: string;
  I: Integer;
begin
  ColList := '';
  ParamList := '';
  
  for I := 0 to High(AColumns) do
  begin
    if I > 0 then begin ColList := ColList + ','; ParamList := ParamList + ','; end;
    ColList := ColList + AColumns[I];
    ParamList := ParamList + ':p' + IntToStr(I);
  end;
  
  SQL := Format('INSERT INTO %s (%s) VALUES (%s)', 
    [ATable, ColList, ParamList]);
  
  FTransaction.StartTransaction;
  try
    FQuery.SQL.Text := SQL;
    FQuery.Prepare;
    for I := 0 to High(AValues) do
      FQuery.Params[I].Value := AValues[I];
    FQuery.ExecSQL;
    
    { Get Last Insert ID }
    FQuery.SQL.Text := 'SELECT last_insert_rowid()';
    FQuery.Open;
    Result := FQuery.Fields[0].AsInteger;
    FQuery.Close;
    
    FTransaction.Commit;
  except
    FTransaction.Rollback;
    raise;
  end;
end;

function TSecureQuery.Update_(const ATable: string;
  const ASetColumns: array of string;
  const ASetValues: array of Variant;
  const AWhere: string;
  const AWhereParams: array of Variant): Integer;
var
  SQL, SetClause: string;
  I: Integer;
  AllParams: array of Variant;
begin
  SetClause := '';
  for I := 0 to High(ASetColumns) do
  begin
    if I > 0 then SetClause := SetClause + ', ';
    SetClause := SetClause + ASetColumns[I] + ' = :s' + IntToStr(I);
  end;
  
  SQL := Format('UPDATE %s SET %s WHERE %s', [ATable, SetClause, AWhere]);
  
  { รวม Parameters }
  SetLength(AllParams, Length(ASetValues) + Length(AWhereParams));
  for I := 0 to High(ASetValues) do
    AllParams[I] := ASetValues[I];
  for I := 0 to High(AWhereParams) do
    AllParams[Length(ASetValues) + I] := AWhereParams[I];
  
  PrepareAndExecute(SQL, AllParams);
  Result := FQuery.RowsAffected;
end;

function TSecureQuery.Delete_(const ATable, AWhere: string;
  const AWhereParams: array of Variant): Integer;
var
  SQL: string;
begin
  SQL := Format('DELETE FROM %s WHERE %s', [ATable, AWhere]);
  PrepareAndExecute(SQL, AWhereParams);
  Result := FQuery.RowsAffected;
end;

end.
```

---

## 4. Secure Login System

```pascal
unit SecureLoginSystem;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, PasswordHashing, InputValidation, SecureDatabase;

type
  TLoginResult = record
    Success: Boolean;
    UserID: Integer;
    Username: string;
    Role: string;
    Token: string;
    ErrorMessage: string;
    RemainingAttempts: Integer;
    LockoutSeconds: Integer;
  end;

  TSecureLoginSystem = class
  private
    FDB: TSecureQuery;
    FRateLimiter: TLoginRateLimiter;
    FRequireMFA: Boolean;
    
    function GenerateToken(AUserID: Integer; const AUsername: string): string;
    function CreateAuditLog(const AEvent, AUsername, AIP, AUserAgent: string; ASuccess: Boolean): void;
    procedure SendSecurityAlert(const AUsername, AIP, AReason: string);
  public
    constructor Create(ADB: TSecureQuery);
    destructor Destroy; override;
    
    function Login(const AUsername, APassword, AIP, AUserAgent: string): TLoginResult;
    function Register(const AUsername, APassword, AEmail: string): TLoginResult;
    function ChangePassword(AUserID: Integer; const AOldPass, ANewPass: string): Boolean;
    function ResetPassword(const AEmail: string): Boolean;
    function Logout(const AToken: string): Boolean;
    function ValidateToken(const AToken: string; out AUserID: Integer): Boolean;
    
    property RequireMFA: Boolean read FRequireMFA write FRequireMFA;
  end;

implementation

constructor TSecureLoginSystem.Create(ADB: TSecureQuery);
begin
  FDB := ADB;
  FRateLimiter := TLoginRateLimiter.Create;
  FRateLimiter.MaxAttempts := 5;
  FRateLimiter.WindowSeconds := 300;
  FRateLimiter.LockoutSeconds := 900;
  FRequireMFA := False;
end;

destructor TSecureLoginSystem.Destroy;
begin
  FRateLimiter.Free;
  inherited;
end;

function TSecureLoginSystem.GenerateToken(AUserID: Integer; const AUsername: string): string;
begin
  { สร้าง Secure Token โดยใช้ Random + User Data }
  Result := Format('%s_%d_%s_%s',
    [FormatDateTime('yyyymmddhhnnss', Now),
     AUserID,
     AUsername,
     IntToHex(Random($FFFFFFFF), 8) + IntToHex(Random($FFFFFFFF), 8)]);
end;

function TSecureLoginSystem.Login(const AUsername, APassword, AIP, AUserAgent: string): TLoginResult;
var
  Validation: TValidationResult;
  StoredHash: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  { 1. Validate Input }
  Validation := TInputValidator.ValidateUsername(AUsername);
  if not Validation.IsValid then
  begin
    Result.Success := False;
    Result.ErrorMessage := Validation.ErrorMessage;
    Exit;
  end;
  
  { 2. Rate Limiting }
  if not FRateLimiter.IsAllowed(AIP, AUsername) then
  begin
    Result.Success := False;
    Result.LockoutSeconds := FRateLimiter.GetLockoutRemainingSeconds(AIP, AUsername);
    Result.ErrorMessage := Format(
      'บัญชีถูกล็อกชั่วคราว กรุณารอ %d วินาที', [Result.LockoutSeconds]);
    CreateAuditLog('LOGIN_LOCKOUT', AUsername, AIP, AUserAgent, False);
    Exit;
  end;
  
  { 3. Find User in Database (Using Parameterized Query) }
  if not FDB.SelectOne('users', 'username = :p0 AND is_active = 1', [AUsername]) then
  begin
    { ไม่พบผู้ใช้ แต่ยังคง Verify เพื่อป้องกัน Timing Attack }
    TPasswordHasher.VerifyPassword(APassword, 
      '$v1$10000$fakesalt123456789012345678901234$fakehash1234567890abcdef1234567890abcdef12345678');
    
    FRateLimiter.RecordAttempt(AIP, AUsername, False);
    Result.Success := False;
    Result.ErrorMessage := 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง';
    Result.RemainingAttempts := FRateLimiter.GetRemainingAttempts(AIP, AUsername);
    
    CreateAuditLog('LOGIN_FAIL', AUsername, AIP, AUserAgent, False);
    Exit;
  end;
  
  StoredHash := FDB.Query.FieldByName('password_hash').AsString;
  
  { 4. Verify Password }
  if not TPasswordHasher.VerifyPassword(APassword, StoredHash) then
  begin
    FRateLimiter.RecordAttempt(AIP, AUsername, False);
    Result.Success := False;
    Result.ErrorMessage := 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง';
    Result.RemainingAttempts := FRateLimiter.GetRemainingAttempts(AIP, AUsername);
    
    CreateAuditLog('LOGIN_FAIL', AUsername, AIP, AUserAgent, False);
    
    { ส่ง Alert เมื่อใกล้จะ Lockout }
    if Result.RemainingAttempts <= 1 then
      SendSecurityAlert(AUsername, AIP, 'Multiple failed login attempts');
    
    Exit;
  end;
  
  { 5. Success }
  FRateLimiter.RecordAttempt(AIP, AUsername, True);
  FRateLimiter.Reset(AIP, AUsername);
  
  Result.Success := True;
  Result.UserID := FDB.Query.FieldByName('id').AsInteger;
  Result.Username := AUsername;
  Result.Role := FDB.Query.FieldByName('role').AsString;
  Result.Token := GenerateToken(Result.UserID, AUsername);
  
  { 6. Re-hash if needed }
  if TPasswordHasher.NeedsRehash(StoredHash) then
  begin
    var NewHash := TPasswordHasher.HashPassword(APassword);
    FDB.Update_('users', ['password_hash'], [NewHash], 
                'id = :p0', [Result.UserID]);
  end;
  
  { 7. บันทึก Session Token }
  FDB.Insert('sessions', 
    ['user_id', 'token', 'ip', 'user_agent', 'created_at', 'expires_at'],
    [Result.UserID, Result.Token, AIP, AUserAgent, 
     FormatDateTime('yyyy-mm-dd hh:nn:ss', Now),
     FormatDateTime('yyyy-mm-dd hh:nn:ss', Now + 1)]);  { Expire ใน 1 วัน }
  
  CreateAuditLog('LOGIN_SUCCESS', AUsername, AIP, AUserAgent, True);
end;

function TSecureLoginSystem.Register(const AUsername, APassword, AEmail: string): TLoginResult;
var
  UsernameValidation, PasswordValidation, EmailValidation: TValidationResult;
  PasswordHash: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  
  { Validate all inputs }
  UsernameValidation := TInputValidator.ValidateUsername(AUsername);
  PasswordValidation := TInputValidator.ValidatePassword(APassword);
  EmailValidation := TInputValidator.ValidateEmail(AEmail);
  
  if not UsernameValidation.IsValid then
  begin
    Result.ErrorMessage := UsernameValidation.ErrorMessage;
    Exit;
  end;
  
  if not PasswordValidation.IsValid then
  begin
    Result.ErrorMessage := PasswordValidation.ErrorMessage;
    Exit;
  end;
  
  if not EmailValidation.IsValid then
  begin
    Result.ErrorMessage := EmailValidation.ErrorMessage;
    Exit;
  end;
  
  { ตรวจสอบว่าชื่อไม่ซ้ำ }
  if FDB.SelectOne('users', 'username = :p0', [LowerCase(AUsername)]) then
  begin
    Result.ErrorMessage := 'ชื่อผู้ใช้นี้ถูกใช้งานแล้ว';
    Exit;
  end;
  
  { ตรวจสอบว่าอีเมลไม่ซ้ำ }
  if FDB.SelectOne('users', 'email = :p0', [LowerCase(AEmail)]) then
  begin
    Result.ErrorMessage := 'อีเมลนี้ถูกใช้งานแล้ว';
    Exit;
  end;
  
  { Hash Password }
  PasswordHash := TPasswordHasher.HashPassword(APassword);
  
  { บันทึกในฐานข้อมูล }
  var UserID := FDB.Insert('users',
    ['username', 'password_hash', 'email', 'role', 'is_active', 'created_at'],
    [LowerCase(AUsername), PasswordHash, LowerCase(AEmail), 'user', True,
     FormatDateTime('yyyy-mm-dd hh:nn:ss', Now)]);
  
  Result.Success := True;
  Result.UserID := UserID;
  Result.Username := AUsername;
  Result.Role := 'user';
end;

function TSecureLoginSystem.ChangePassword(AUserID: Integer; 
  const AOldPass, ANewPass: string): Boolean;
var
  StoredHash: string;
  Validation: TValidationResult;
begin
  Result := False;
  
  { Validate new password }
  Validation := TInputValidator.ValidatePassword(ANewPass);
  if not Validation.IsValid then
    raise Exception.Create(Validation.ErrorMessage);
  
  { ตรวจสอบรหัสผ่านเก่า }
  if not FDB.SelectOne('users', 'id = :p0', [AUserID]) then
    raise Exception.Create('ไม่พบผู้ใช้');
  
  StoredHash := FDB.Query.FieldByName('password_hash').AsString;
  
  if not TPasswordHasher.VerifyPassword(AOldPass, StoredHash) then
    raise Exception.Create('รหัสผ่านเดิมไม่ถูกต้อง');
  
  { ตรวจสอบว่าไม่เหมือนรหัสผ่านเก่า }
  if TPasswordHasher.VerifyPassword(ANewPass, StoredHash) then
    raise Exception.Create('รหัสผ่านใหม่ต้องไม่เหมือนรหัสผ่านเดิม');
  
  var NewHash := TPasswordHasher.HashPassword(ANewPass);
  FDB.Update_('users', ['password_hash'], [NewHash], 'id = :p0', [AUserID]);
  
  { ล้าง Sessions ทั้งหมดเพื่อ Force Re-login }
  FDB.Delete_('sessions', 'user_id = :p0', [AUserID]);
  
  Result := True;
end;

function TSecureLoginSystem.ResetPassword(const AEmail: string): Boolean;
var
  ResetToken: string;
begin
  Result := False;
  
  { ตรวจสอบอีเมล }
  if not FDB.SelectOne('users', 'email = :p0 AND is_active = 1', 
                        [LowerCase(AEmail)]) then
  begin
    { ไม่แจ้งว่าไม่พบอีเมล เพื่อป้องกัน User Enumeration }
    Result := True;
    Exit;
  end;
  
  ResetToken := IntToHex(Random($FFFFFFFF), 8) + IntToHex(Random($FFFFFFFF), 8);
  
  var UserID := FDB.Query.FieldByName('id').AsInteger;
  
  { บันทึก Reset Token }
  FDB.Insert('password_resets',
    ['user_id', 'token', 'created_at', 'expires_at', 'used'],
    [UserID, ResetToken,
     FormatDateTime('yyyy-mm-dd hh:nn:ss', Now),
     FormatDateTime('yyyy-mm-dd hh:nn:ss', Now + 1/24),  { Expire ใน 1 ชั่วโมง }
     False]);
  
  { TODO: ส่งอีเมล Reset Password }
  WriteLn('ส่งอีเมล Reset Password ไปยัง ' + AEmail);
  WriteLn('Token: ' + ResetToken);
  
  Result := True;
end;

function TSecureLoginSystem.Logout(const AToken: string): Boolean;
begin
  Result := FDB.Delete_('sessions', 'token = :p0', [AToken]) > 0;
end;

function TSecureLoginSystem.ValidateToken(const AToken: string; out AUserID: Integer): Boolean;
begin
  AUserID := 0;
  
  if not FDB.SelectOne('sessions', 
    'token = :p0 AND expires_at > :p1',
    [AToken, FormatDateTime('yyyy-mm-dd hh:nn:ss', Now)]) then
  begin
    Result := False;
    Exit;
  end;
  
  AUserID := FDB.Query.FieldByName('user_id').AsInteger;
  Result := True;
end;

function TSecureLoginSystem.CreateAuditLog(const AEvent, AUsername, AIP, 
  AUserAgent: string; ASuccess: Boolean): void;
begin
  try
    FDB.Insert('audit_logs',
      ['event', 'username', 'ip', 'user_agent', 'success', 'created_at'],
      [AEvent, AUsername, AIP, AUserAgent, ASuccess,
       FormatDateTime('yyyy-mm-dd hh:nn:ss', Now)]);
  except
    { ไม่ให้ Log Error หยุดการทำงาน }
  end;
end;

procedure TSecureLoginSystem.SendSecurityAlert(const AUsername, AIP, AReason: string);
begin
  { TODO: Implement Email/SMS Alert }
  WriteLn(Format('SECURITY ALERT: User=%s, IP=%s, Reason=%s', [AUsername, AIP, AReason]));
end;

end.
```

---

## แบบฝึกหัด

### ข้อ 1 - SQL Injection Test
เขียนโปรแกรมทดสอบ SQL Injection ที่รองรับการตรวจสอบ Input ที่อันตรายทั้งหมดและบันทึกลง Log

### ข้อ 2 - Password Strength Checker
สร้าง Password Strength Checker ที่แสดง Strength เป็น: Weak, Fair, Good, Strong, Very Strong

### ข้อ 3 - Secure File Upload
สร้างระบบอัปโหลดไฟล์ที่ปลอดภัย:
- ตรวจสอบ MIME Type จากเนื้อหาไฟล์ ไม่ใช่นามสกุล
- สร้างชื่อไฟล์ใหม่แบบ Random
- สแกนหา Malware (จำลอง)

### ข้อ 4 - CSRF Protection
สร้าง CSRF Token Manager:
- Generate Token
- Validate Token
- Per-session Tokens

### ข้อ 5 - Security Audit Logger
สร้าง Security Audit Logger ที่บันทึก:
- ทุก Login/Logout
- การเปลี่ยนรหัสผ่าน
- การพยายาม SQL Injection
- Access ที่ Unauthorized
