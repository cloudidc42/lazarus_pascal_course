# ตอนที่ 74: Configuration Management ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการจัดการ configuration ในรูปแบบต่างๆ ตั้งแต่ INI files, JSON, environment variables ไปจนถึง command-line arguments

---

## 74.1 INI Configuration

```pascal
unit config_ini;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, IniFiles;

type
  TDatabaseConfig = record
    Host: string;
    Port: Integer;
    Database: string;
    Username: string;
    Password: string;
    MaxConnections: Integer;
    Timeout: Integer;
    SSL: Boolean;
  end;

  TServerConfig = record
    Host: string;
    Port: Integer;
    MaxClients: Integer;
    ReadTimeout: Integer;
    WriteTimeout: Integer;
    EnableHTTPS: Boolean;
    CertFile: string;
    KeyFile: string;
  end;

  TAppConfig = record
    AppName: string;
    Version: string;
    Debug: Boolean;
    LogLevel: string;
    LogFile: string;
    TempDir: string;
    MaxUploadSize: Int64;
    Database: TDatabaseConfig;
    Server: TServerConfig;
  end;

  TINIConfig = class
  private
    FINI: TINIFile;
    FConfig: TAppConfig;
    FFilePath: string;
    FIsLoaded: Boolean;
    
    procedure LoadDatabase;
    procedure LoadServer;
    procedure LoadApp;
    procedure SaveDatabase;
    procedure SaveServer;
    procedure SaveApp;
    
  public
    constructor Create(const AFilePath: string);
    destructor Destroy; override;
    
    function Load: Boolean;
    procedure Save;
    procedure SetDefaults;
    
    function GetString(const ASection, AKey, ADefault: string): string;
    function GetInteger(const ASection, AKey: string; ADefault: Integer): Integer;
    function GetBoolean(const ASection, AKey: string; ADefault: Boolean): Boolean;
    function GetInt64(const ASection, AKey: string; ADefault: Int64): Int64;
    
    procedure SetValue(const ASection, AKey, AValue: string);
    
    property Config: TAppConfig read FConfig write FConfig;
    property FilePath: string read FFilePath;
    property IsLoaded: Boolean read FIsLoaded;
  end;

implementation

constructor TINIConfig.Create(const AFilePath: string);
begin
  inherited Create;
  FFilePath := AFilePath;
  FIsLoaded := False;
  SetDefaults;
end;

destructor TINIConfig.Destroy;
begin
  if Assigned(FINI) then
    FINI.Free;
  inherited Destroy;
end;

procedure TINIConfig.SetDefaults;
begin
  // App defaults
  FConfig.AppName := 'MyApplication';
  FConfig.Version := '1.0.0';
  FConfig.Debug := False;
  FConfig.LogLevel := 'info';
  FConfig.LogFile := '/var/log/myapp/app.log';
  FConfig.TempDir := '/tmp/myapp';
  FConfig.MaxUploadSize := 10 * 1024 * 1024;  // 10MB
  
  // Database defaults
  FConfig.Database.Host := 'localhost';
  FConfig.Database.Port := 5432;
  FConfig.Database.Database := 'mydb';
  FConfig.Database.Username := 'postgres';
  FConfig.Database.Password := '';
  FConfig.Database.MaxConnections := 10;
  FConfig.Database.Timeout := 30;
  FConfig.Database.SSL := False;
  
  // Server defaults
  FConfig.Server.Host := '0.0.0.0';
  FConfig.Server.Port := 8080;
  FConfig.Server.MaxClients := 100;
  FConfig.Server.ReadTimeout := 60;
  FConfig.Server.WriteTimeout := 60;
  FConfig.Server.EnableHTTPS := False;
  FConfig.Server.CertFile := '';
  FConfig.Server.KeyFile := '';
end;

function TINIConfig.GetString(const ASection, AKey, ADefault: string): string;
begin
  if Assigned(FINI) then
    Result := FINI.ReadString(ASection, AKey, ADefault)
  else
    Result := ADefault;
end;

function TINIConfig.GetInteger(const ASection, AKey: string; ADefault: Integer): Integer;
begin
  if Assigned(FINI) then
    Result := FINI.ReadInteger(ASection, AKey, ADefault)
  else
    Result := ADefault;
end;

function TINIConfig.GetBoolean(const ASection, AKey: string; ADefault: Boolean): Boolean;
begin
  if Assigned(FINI) then
    Result := FINI.ReadBool(ASection, AKey, ADefault)
  else
    Result := ADefault;
end;

function TINIConfig.GetInt64(const ASection, AKey: string; ADefault: Int64): Int64;
begin
  if Assigned(FINI) then
    Result := FINI.ReadInt64(ASection, AKey, ADefault)
  else
    Result := ADefault;
end;

procedure TINIConfig.SetValue(const ASection, AKey, AValue: string);
begin
  if Assigned(FINI) then
    FINI.WriteString(ASection, AKey, AValue);
end;

procedure TINIConfig.LoadDatabase;
begin
  FConfig.Database.Host := GetString('Database', 'Host', FConfig.Database.Host);
  FConfig.Database.Port := GetInteger('Database', 'Port', FConfig.Database.Port);
  FConfig.Database.Database := GetString('Database', 'Database', FConfig.Database.Database);
  FConfig.Database.Username := GetString('Database', 'Username', FConfig.Database.Username);
  FConfig.Database.Password := GetString('Database', 'Password', FConfig.Database.Password);
  FConfig.Database.MaxConnections := GetInteger('Database', 'MaxConnections', 
    FConfig.Database.MaxConnections);
  FConfig.Database.Timeout := GetInteger('Database', 'Timeout', FConfig.Database.Timeout);
  FConfig.Database.SSL := GetBoolean('Database', 'SSL', FConfig.Database.SSL);
end;

procedure TINIConfig.LoadServer;
begin
  FConfig.Server.Host := GetString('Server', 'Host', FConfig.Server.Host);
  FConfig.Server.Port := GetInteger('Server', 'Port', FConfig.Server.Port);
  FConfig.Server.MaxClients := GetInteger('Server', 'MaxClients', FConfig.Server.MaxClients);
  FConfig.Server.ReadTimeout := GetInteger('Server', 'ReadTimeout', FConfig.Server.ReadTimeout);
  FConfig.Server.WriteTimeout := GetInteger('Server', 'WriteTimeout', FConfig.Server.WriteTimeout);
  FConfig.Server.EnableHTTPS := GetBoolean('Server', 'EnableHTTPS', FConfig.Server.EnableHTTPS);
  FConfig.Server.CertFile := GetString('Server', 'CertFile', FConfig.Server.CertFile);
  FConfig.Server.KeyFile := GetString('Server', 'KeyFile', FConfig.Server.KeyFile);
end;

procedure TINIConfig.LoadApp;
begin
  FConfig.AppName := GetString('Application', 'Name', FConfig.AppName);
  FConfig.Version := GetString('Application', 'Version', FConfig.Version);
  FConfig.Debug := GetBoolean('Application', 'Debug', FConfig.Debug);
  FConfig.LogLevel := GetString('Application', 'LogLevel', FConfig.LogLevel);
  FConfig.LogFile := GetString('Application', 'LogFile', FConfig.LogFile);
  FConfig.TempDir := GetString('Application', 'TempDir', FConfig.TempDir);
  FConfig.MaxUploadSize := GetInt64('Application', 'MaxUploadSize', FConfig.MaxUploadSize);
end;

function TINIConfig.Load: Boolean;
begin
  Result := False;
  
  if not FileExists(FFilePath) then
  begin
    WriteLn('Config file not found: ', FFilePath);
    WriteLn('Using defaults');
    Result := True;  // ใช้ defaults ได้
    FIsLoaded := True;
    Exit;
  end;
  
  try
    FINI := TINIFile.Create(FFilePath);
    
    LoadApp;
    LoadDatabase;
    LoadServer;
    
    FIsLoaded := True;
    Result := True;
    WriteLn('Configuration loaded from: ', FFilePath);
    
  except
    on E: Exception do
    begin
      WriteLn('Error loading config: ', E.Message);
      FIsLoaded := False;
    end;
  end;
end;

procedure TINIConfig.SaveDatabase;
begin
  FINI.WriteString('Database', 'Host', FConfig.Database.Host);
  FINI.WriteInteger('Database', 'Port', FConfig.Database.Port);
  FINI.WriteString('Database', 'Database', FConfig.Database.Database);
  FINI.WriteString('Database', 'Username', FConfig.Database.Username);
  FINI.WriteString('Database', 'Password', FConfig.Database.Password);
  FINI.WriteInteger('Database', 'MaxConnections', FConfig.Database.MaxConnections);
  FINI.WriteInteger('Database', 'Timeout', FConfig.Database.Timeout);
  FINI.WriteBool('Database', 'SSL', FConfig.Database.SSL);
end;

procedure TINIConfig.SaveServer;
begin
  FINI.WriteString('Server', 'Host', FConfig.Server.Host);
  FINI.WriteInteger('Server', 'Port', FConfig.Server.Port);
  FINI.WriteInteger('Server', 'MaxClients', FConfig.Server.MaxClients);
  FINI.WriteInteger('Server', 'ReadTimeout', FConfig.Server.ReadTimeout);
  FINI.WriteInteger('Server', 'WriteTimeout', FConfig.Server.WriteTimeout);
  FINI.WriteBool('Server', 'EnableHTTPS', FConfig.Server.EnableHTTPS);
  FINI.WriteString('Server', 'CertFile', FConfig.Server.CertFile);
  FINI.WriteString('Server', 'KeyFile', FConfig.Server.KeyFile);
end;

procedure TINIConfig.SaveApp;
begin
  FINI.WriteString('Application', 'Name', FConfig.AppName);
  FINI.WriteString('Application', 'Version', FConfig.Version);
  FINI.WriteBool('Application', 'Debug', FConfig.Debug);
  FINI.WriteString('Application', 'LogLevel', FConfig.LogLevel);
  FINI.WriteString('Application', 'LogFile', FConfig.LogFile);
  FINI.WriteString('Application', 'TempDir', FConfig.TempDir);
  FINI.WriteInt64('Application', 'MaxUploadSize', FConfig.MaxUploadSize);
end;

procedure TINIConfig.Save;
begin
  if not Assigned(FINI) then
    FINI := TINIFile.Create(FFilePath);
    
  SaveApp;
  SaveDatabase;
  SaveServer;
  
  WriteLn('Configuration saved to: ', FFilePath);
end;

end.
```

---

## 74.2 JSON Configuration

```pascal
unit config_json;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fpjson, jsonparser;

type
  TJSONConfig = class
  private
    FFilePath: string;
    FData: TJSONObject;
    FIsLoaded: Boolean;
    
    function GetOrCreate(const APath: string): TJSONData;
    function NavigatePath(const APath: string; ACreate: Boolean = False): TJSONObject;
    
  public
    constructor Create(const AFilePath: string);
    destructor Destroy; override;
    
    function Load: Boolean;
    procedure Save;
    
    function GetString(const APath, ADefault: string): string;
    function GetInteger(const APath: string; ADefault: Integer): Integer;
    function GetBoolean(const APath: string; ADefault: Boolean): Boolean;
    function GetFloat(const APath: string; ADefault: Double): Double;
    function GetArray(const APath: string): TJSONArray;
    
    procedure SetString(const APath, AValue: string);
    procedure SetInteger(const APath: string; AValue: Integer);
    procedure SetBoolean(const APath: string; AValue: Boolean);
    
    function ToJSON: string;
    
    property FilePath: string read FFilePath;
    property IsLoaded: Boolean read FIsLoaded;
    property Data: TJSONObject read FData;
  end;

implementation

constructor TJSONConfig.Create(const AFilePath: string);
begin
  inherited Create;
  FFilePath := AFilePath;
  FData := TJSONObject.Create;
  FIsLoaded := False;
end;

destructor TJSONConfig.Destroy;
begin
  FData.Free;
  inherited Destroy;
end;

function TJSONConfig.Load: Boolean;
var
  Content: string;
  F: TextFile;
  Line: string;
  Parser: TJSONParser;
  Parsed: TJSONData;
begin
  Result := False;
  
  if not FileExists(FFilePath) then
  begin
    WriteLn('JSON config not found, using defaults: ', FFilePath);
    FIsLoaded := True;
    Result := True;
    Exit;
  end;
  
  try
    Content := '';
    AssignFile(F, FFilePath);
    Reset(F);
    try
      while not Eof(F) do
      begin
        ReadLn(F, Line);
        Content := Content + Line + LineEnding;
      end;
    finally
      CloseFile(F);
    end;
    
    Parser := TJSONParser.Create(Content, []);
    try
      Parsed := Parser.Parse;
      
      if Parsed is TJSONObject then
      begin
        FData.Free;
        FData := TJSONObject(Parsed);
        FIsLoaded := True;
        Result := True;
        WriteLn('JSON config loaded: ', FFilePath);
      end
      else
      begin
        Parsed.Free;
        WriteLn('Invalid JSON config format');
      end;
    finally
      Parser.Free;
    end;
    
  except
    on E: Exception do
      WriteLn('Error loading JSON config: ', E.Message);
  end;
end;

procedure TJSONConfig.Save;
var
  F: TextFile;
  Content: string;
begin
  try
    Content := FData.FormatJSON([foDoNotQuoteMembers, foSingleLineArray], 2);
    
    AssignFile(F, FFilePath);
    Rewrite(F);
    try
      WriteLn(F, Content);
    finally
      CloseFile(F);
    end;
    
    WriteLn('JSON config saved: ', FFilePath);
  except
    on E: Exception do
      WriteLn('Error saving JSON config: ', E.Message);
  end;
end;

function TJSONConfig.NavigatePath(const APath: string; ACreate: Boolean): TJSONObject;
var
  Parts: TStringList;
  i: Integer;
  Current: TJSONObject;
  Child: TJSONData;
begin
  Result := FData;
  
  if APath = '' then Exit;
  
  Parts := TStringList.Create;
  try
    Parts.Delimiter := '.';
    Parts.DelimitedText := APath;
    
    Current := FData;
    
    for i := 0 to Parts.Count - 2 do  // Navigate all but last
    begin
      Child := Current.Find(Parts[i]);
      
      if Child = nil then
      begin
        if ACreate then
        begin
          var NewObj := TJSONObject.Create;
          Current.Add(Parts[i], NewObj);
          Current := NewObj;
        end
        else
        begin
          Result := nil;
          Exit;
        end;
      end
      else if Child is TJSONObject then
        Current := TJSONObject(Child)
      else
      begin
        Result := nil;
        Exit;
      end;
    end;
    
    Result := Current;
    
  finally
    Parts.Free;
  end;
end;

function TJSONConfig.GetString(const APath, ADefault: string): string;
var
  Parts: TStringList;
  Obj: TJSONObject;
  Key: string;
  Val: TJSONData;
begin
  Result := ADefault;
  
  Parts := TStringList.Create;
  try
    Parts.Delimiter := '.';
    Parts.DelimitedText := APath;
    
    if Parts.Count = 0 then Exit;
    
    Key := Parts[Parts.Count - 1];
    Parts.Delete(Parts.Count - 1);
    
    Obj := NavigatePath(Parts.DelimitedText);
    if Obj = nil then Exit;
    
    Val := Obj.Find(Key);
    if Val <> nil then
      Result := Val.AsString;
  finally
    Parts.Free;
  end;
end;

function TJSONConfig.GetInteger(const APath: string; ADefault: Integer): Integer;
var
  StrVal: string;
begin
  StrVal := GetString(APath, '');
  if StrVal = '' then
    Result := ADefault
  else
    Result := StrToIntDef(StrVal, ADefault);
end;

function TJSONConfig.GetBoolean(const APath: string; ADefault: Boolean): Boolean;
var
  StrVal: string;
begin
  StrVal := LowerCase(GetString(APath, ''));
  if StrVal = '' then
    Result := ADefault
  else
    Result := (StrVal = 'true') or (StrVal = '1') or (StrVal = 'yes');
end;

function TJSONConfig.GetFloat(const APath: string; ADefault: Double): Double;
var
  StrVal: string;
begin
  StrVal := GetString(APath, '');
  if StrVal = '' then
    Result := ADefault
  else
    Result := StrToFloatDef(StrVal, ADefault);
end;

procedure TJSONConfig.SetString(const APath, AValue: string);
var
  Parts: TStringList;
  Obj: TJSONObject;
  Key: string;
begin
  Parts := TStringList.Create;
  try
    Parts.Delimiter := '.';
    Parts.DelimitedText := APath;
    
    if Parts.Count = 0 then Exit;
    
    Key := Parts[Parts.Count - 1];
    Parts.Delete(Parts.Count - 1);
    
    Obj := NavigatePath(Parts.DelimitedText, True);
    if Obj = nil then Exit;
    
    var Existing := Obj.Find(Key);
    if Existing <> nil then
    begin
      Obj.Remove(Existing);
      Existing.Free;
    end;
    
    Obj.Add(Key, TJSONString.Create(AValue));
  finally
    Parts.Free;
  end;
end;

procedure TJSONConfig.SetInteger(const APath: string; AValue: Integer);
begin
  SetString(APath, IntToStr(AValue));
end;

procedure TJSONConfig.SetBoolean(const APath: string; AValue: Boolean);
begin
  SetString(APath, IfThen(AValue, 'true', 'false'));
end;

function TJSONConfig.GetArray(const APath: string): TJSONArray;
var
  Parts: TStringList;
  Obj: TJSONObject;
  Key: string;
  Val: TJSONData;
begin
  Result := nil;
  
  Parts := TStringList.Create;
  try
    Parts.Delimiter := '.';
    Parts.DelimitedText := APath;
    
    if Parts.Count = 0 then Exit;
    
    Key := Parts[Parts.Count - 1];
    Parts.Delete(Parts.Count - 1);
    
    Obj := NavigatePath(Parts.DelimitedText);
    if Obj = nil then Exit;
    
    Val := Obj.Find(Key);
    if Val is TJSONArray then
      Result := TJSONArray(Val);
  finally
    Parts.Free;
  end;
end;

function TJSONConfig.ToJSON: string;
begin
  Result := FData.FormatJSON([foDoNotQuoteMembers], 2);
end;

end.
```

---

## 74.3 Environment Variables & Command-line

```pascal
unit config_env;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TEnvConfig = class
  private
    FPrefix: string;
    FValues: TStringList;
    
    function BuildKey(const AKey: string): string;
    
  public
    constructor Create(const APrefix: string = '');
    destructor Destroy; override;
    
    procedure LoadFromEnvironment;
    
    function GetString(const AKey, ADefault: string): string;
    function GetInteger(const AKey: string; ADefault: Integer): Integer;
    function GetBoolean(const AKey: string; ADefault: Boolean): Boolean;
    function GetFloat(const AKey: string; ADefault: Double): Double;
    
    function HasKey(const AKey: string): Boolean;
    procedure DumpAll;
    
    property Prefix: string read FPrefix;
  end;

  // Command-line argument parser
  TArgType = (atFlag, atString, atInteger, atFloat);
  
  TArgDef = record
    ShortName: Char;
    LongName: string;
    ArgType: TArgType;
    Default: string;
    Description: string;
    Required: Boolean;
  end;

  TArgParser = class
  private
    FDefs: array of TArgDef;
    FDefCount: Integer;
    FValues: TStringList;
    FPositional: TStringList;
    FProgramName: string;
    
    procedure ParseArgs(const AArgs: array of string);
    function FindDef(const AName: string): Integer;
    
  public
    constructor Create(const AProgramName: string = '');
    destructor Destroy; override;
    
    procedure AddFlag(AShort: Char; const ALong, ADescription: string);
    procedure AddString(AShort: Char; const ALong, ADefault, ADescription: string; 
      ARequired: Boolean = False);
    procedure AddInteger(AShort: Char; const ALong: string; ADefault: Integer; 
      const ADescription: string; ARequired: Boolean = False);
    
    function Parse: Boolean;  // Parse from command line
    function ParseFrom(const AArgs: array of string): Boolean;
    
    function GetFlag(const AName: string): Boolean;
    function GetString(const AName, ADefault: string = ''): string;
    function GetInteger(const AName: string; ADefault: Integer = 0): Integer;
    function GetPositional(AIndex: Integer): string;
    function PositionalCount: Integer;
    
    procedure PrintHelp;
    procedure PrintVersion(const AVersion: string);
    
    property ProgramName: string read FProgramName write FProgramName;
  end;

implementation

{ TEnvConfig }

constructor TEnvConfig.Create(const APrefix: string);
begin
  inherited Create;
  FPrefix := UpperCase(APrefix);
  FValues := TStringList.Create;
end;

destructor TEnvConfig.Destroy;
begin
  FValues.Free;
  inherited Destroy;
end;

function TEnvConfig.BuildKey(const AKey: string): string;
begin
  if FPrefix <> '' then
    Result := FPrefix + '_' + UpperCase(AKey)
  else
    Result := UpperCase(AKey);
end;

procedure TEnvConfig.LoadFromEnvironment;
var
  i: Integer;
  EnvKey, EnvVal: string;
begin
  FValues.Clear;
  
  // บน Linux/Mac ใช้ fpGetEnviron
  {$IFDEF UNIX}
  var EnvCount := 0;
  var P := fpGetEnviron;
  while P^ <> nil do
  begin
    var S := String(P^);
    var EqPos := Pos('=', S);
    if EqPos > 0 then
    begin
      EnvKey := Copy(S, 1, EqPos - 1);
      EnvVal := Copy(S, EqPos + 1, MaxInt);
      
      // Filter by prefix
      if (FPrefix = '') or (Copy(EnvKey, 1, Length(FPrefix) + 1) = FPrefix + '_') then
        FValues.Values[EnvKey] := EnvVal;
    end;
    Inc(P);
    Inc(EnvCount);
  end;
  {$ENDIF}
  
  {$IFDEF WINDOWS}
  // Windows environment variables
  var Env := GetEnvironmentStrings;
  var P := Env;
  while P^ <> #0 do
  begin
    var S := StrPas(P);
    var EqPos := Pos('=', S);
    if EqPos > 0 then
    begin
      EnvKey := Copy(S, 1, EqPos - 1);
      EnvVal := Copy(S, EqPos + 1, MaxInt);
      FValues.Values[EnvKey] := EnvVal;
    end;
    Inc(P, Length(S) + 1);
  end;
  FreeEnvironmentStrings(Env);
  {$ENDIF}
end;

function TEnvConfig.GetString(const AKey, ADefault: string): string;
var
  FullKey: string;
begin
  FullKey := BuildKey(AKey);
  
  // Check pre-loaded values first
  if FValues.IndexOfName(FullKey) >= 0 then
  begin
    Result := FValues.Values[FullKey];
    Exit;
  end;
  
  // Then try system environment
  Result := GetEnvironmentVariable(FullKey);
  if Result = '' then
    Result := ADefault;
end;

function TEnvConfig.GetInteger(const AKey: string; ADefault: Integer): Integer;
begin
  Result := StrToIntDef(GetString(AKey, ''), ADefault);
end;

function TEnvConfig.GetBoolean(const AKey: string; ADefault: Boolean): Boolean;
var
  Val: string;
begin
  Val := LowerCase(GetString(AKey, ''));
  if Val = '' then
    Result := ADefault
  else
    Result := (Val = 'true') or (Val = '1') or (Val = 'yes') or (Val = 'on');
end;

function TEnvConfig.GetFloat(const AKey: string; ADefault: Double): Double;
begin
  Result := StrToFloatDef(GetString(AKey, ''), ADefault);
end;

function TEnvConfig.HasKey(const AKey: string): Boolean;
begin
  Result := GetEnvironmentVariable(BuildKey(AKey)) <> '';
end;

procedure TEnvConfig.DumpAll;
var
  i: Integer;
  Key: string;
begin
  WriteLn('=== Environment Variables ===');
  if FValues.Count > 0 then
    for i := 0 to FValues.Count - 1 do
    begin
      Key := FValues.Names[i];
      if (FPrefix = '') or (Copy(Key, 1, Length(FPrefix)) = FPrefix) then
        WriteLn(Key, '=', FValues.ValueFromIndex[i]);
    end;
end;

{ TArgParser }

constructor TArgParser.Create(const AProgramName: string);
begin
  inherited Create;
  FProgramName := AProgramName;
  if FProgramName = '' then
    FProgramName := ExtractFileName(ParamStr(0));
  FDefCount := 0;
  SetLength(FDefs, 50);
  FValues := TStringList.Create;
  FPositional := TStringList.Create;
end;

destructor TArgParser.Destroy;
begin
  FValues.Free;
  FPositional.Free;
  inherited Destroy;
end;

procedure TArgParser.AddFlag(AShort: Char; const ALong, ADescription: string);
begin
  if FDefCount >= Length(FDefs) then
    SetLength(FDefs, Length(FDefs) + 20);
    
  FDefs[FDefCount].ShortName := AShort;
  FDefs[FDefCount].LongName := ALong;
  FDefs[FDefCount].ArgType := atFlag;
  FDefs[FDefCount].Default := 'false';
  FDefs[FDefCount].Description := ADescription;
  FDefs[FDefCount].Required := False;
  Inc(FDefCount);
end;

procedure TArgParser.AddString(AShort: Char; const ALong, ADefault, ADescription: string;
  ARequired: Boolean);
begin
  if FDefCount >= Length(FDefs) then
    SetLength(FDefs, Length(FDefs) + 20);
    
  FDefs[FDefCount].ShortName := AShort;
  FDefs[FDefCount].LongName := ALong;
  FDefs[FDefCount].ArgType := atString;
  FDefs[FDefCount].Default := ADefault;
  FDefs[FDefCount].Description := ADescription;
  FDefs[FDefCount].Required := ARequired;
  Inc(FDefCount);
end;

procedure TArgParser.AddInteger(AShort: Char; const ALong: string; ADefault: Integer;
  const ADescription: string; ARequired: Boolean);
begin
  if FDefCount >= Length(FDefs) then
    SetLength(FDefs, Length(FDefs) + 20);
    
  FDefs[FDefCount].ShortName := AShort;
  FDefs[FDefCount].LongName := ALong;
  FDefs[FDefCount].ArgType := atInteger;
  FDefs[FDefCount].Default := IntToStr(ADefault);
  FDefs[FDefCount].Description := ADescription;
  FDefs[FDefCount].Required := ARequired;
  Inc(FDefCount);
end;

function TArgParser.FindDef(const AName: string): Integer;
var
  i: Integer;
begin
  Result := -1;
  for i := 0 to FDefCount - 1 do
  begin
    if (AName = FDefs[i].LongName) or 
       ((Length(AName) = 1) and (AName[1] = FDefs[i].ShortName)) then
    begin
      Result := i;
      Exit;
    end;
  end;
end;

function TArgParser.Parse: Boolean;
var
  Args: array of string;
  i: Integer;
begin
  SetLength(Args, ParamCount);
  for i := 1 to ParamCount do
    Args[i - 1] := ParamStr(i);
  Result := ParseFrom(Args);
end;

function TArgParser.ParseFrom(const AArgs: array of string): Boolean;
var
  i: Integer;
  Arg, Name, Value: string;
  DefIdx: Integer;
  EqPos: Integer;
begin
  Result := True;
  
  // Set defaults first
  for i := 0 to FDefCount - 1 do
    FValues.Values[FDefs[i].LongName] := FDefs[i].Default;
  
  i := 0;
  while i < Length(AArgs) do
  begin
    Arg := AArgs[i];
    
    if Copy(Arg, 1, 2) = '--' then
    begin
      // Long option: --name or --name=value
      Name := Copy(Arg, 3, MaxInt);
      
      EqPos := Pos('=', Name);
      if EqPos > 0 then
      begin
        Value := Copy(Name, EqPos + 1, MaxInt);
        Name := Copy(Name, 1, EqPos - 1);
      end
      else if (i + 1 < Length(AArgs)) and (Copy(AArgs[i + 1], 1, 1) <> '-') then
      begin
        Inc(i);
        Value := AArgs[i];
      end
      else
        Value := 'true';
      
      DefIdx := FindDef(Name);
      if DefIdx >= 0 then
        FValues.Values[FDefs[DefIdx].LongName] := Value
      else
      begin
        WriteLn('Unknown option: --', Name);
        Result := False;
      end;
    end
    else if (Length(Arg) = 2) and (Arg[1] = '-') then
    begin
      // Short option: -x or -x value
      Name := Arg[2];
      DefIdx := FindDef(Name);
      
      if DefIdx >= 0 then
      begin
        if FDefs[DefIdx].ArgType = atFlag then
          FValues.Values[FDefs[DefIdx].LongName] := 'true'
        else if (i + 1 < Length(AArgs)) and (Copy(AArgs[i + 1], 1, 1) <> '-') then
        begin
          Inc(i);
          FValues.Values[FDefs[DefIdx].LongName] := AArgs[i];
        end;
      end
      else
      begin
        WriteLn('Unknown option: -', Name);
        Result := False;
      end;
    end
    else
      FPositional.Add(Arg);
      
    Inc(i);
  end;
  
  // Check required args
  for i := 0 to FDefCount - 1 do
    if FDefs[i].Required and (FValues.Values[FDefs[i].LongName] = '') then
    begin
      WriteLn('Required argument missing: --', FDefs[i].LongName);
      Result := False;
    end;
end;

function TArgParser.GetFlag(const AName: string): Boolean;
begin
  Result := LowerCase(FValues.Values[AName]) = 'true';
end;

function TArgParser.GetString(const AName, ADefault: string): string;
begin
  Result := FValues.Values[AName];
  if Result = '' then Result := ADefault;
end;

function TArgParser.GetInteger(const AName: string; ADefault: Integer): Integer;
begin
  Result := StrToIntDef(FValues.Values[AName], ADefault);
end;

function TArgParser.GetPositional(AIndex: Integer): string;
begin
  if AIndex < FPositional.Count then
    Result := FPositional[AIndex]
  else
    Result := '';
end;

function TArgParser.PositionalCount: Integer;
begin
  Result := FPositional.Count;
end;

procedure TArgParser.PrintHelp;
var
  i: Integer;
  ShortOpt, TypeStr: string;
begin
  WriteLn('Usage: ', FProgramName, ' [options] [arguments]');
  WriteLn('');
  WriteLn('Options:');
  
  for i := 0 to FDefCount - 1 do
  begin
    if FDefs[i].ShortName <> #0 then
      ShortOpt := Format('-%s, ', [FDefs[i].ShortName])
    else
      ShortOpt := '    ';
      
    case FDefs[i].ArgType of
      atFlag:    TypeStr := '';
      atString:  TypeStr := ' <string>';
      atInteger: TypeStr := ' <int>';
      atFloat:   TypeStr := ' <float>';
    end;
    
    WriteLn(Format('  %s--%-20s%s',
      [ShortOpt,
       FDefs[i].LongName + TypeStr,
       FDefs[i].Description]));
       
    if FDefs[i].Default <> '' then
      WriteLn(Format('  %s  Default: %s', [StringOfChar(' ', 26), FDefs[i].Default]));
  end;
end;

end.
```

---

## 74.4 ตัวอย่างการใช้งาน

```pascal
program config_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  config_ini, config_json, config_env;

var
  INI: TINIConfig;
  JSON: TJSONConfig;
  Env: TEnvConfig;
  Args: TArgParser;
begin
  WriteLn('=== Configuration Demo ===');
  
  // 1. INI Config
  INI := TINIConfig.Create('myapp.ini');
  try
    if not INI.Load then
    begin
      WriteLn('Creating default config...');
      INI.Save;
    end;
    
    WriteLn('DB Host: ', INI.Config.Database.Host);
    WriteLn('Server Port: ', INI.Config.Server.Port);
  finally
    INI.Free;
  end;
  
  // 2. JSON Config
  JSON := TJSONConfig.Create('config.json');
  try
    JSON.Load;
    
    var AppName := JSON.GetString('app.name', 'MyApp');
    var Port := JSON.GetInteger('server.port', 8080);
    var Debug := JSON.GetBoolean('app.debug', False);
    
    WriteLn(Format('App: %s, Port: %d, Debug: %s',
      [AppName, Port, IfThen(Debug, 'true', 'false')]));
  finally
    JSON.Free;
  end;
  
  // 3. Environment Variables
  Env := TEnvConfig.Create('MYAPP');
  try
    Env.LoadFromEnvironment;
    
    var DBHost := Env.GetString('DB_HOST', 'localhost');
    var DBPort := Env.GetInteger('DB_PORT', 5432);
    var LogLevel := Env.GetString('LOG_LEVEL', 'info');
    
    WriteLn(Format('DB: %s:%d, Log: %s', [DBHost, DBPort, LogLevel]));
  finally
    Env.Free;
  end;
  
  // 4. Command-line Args
  Args := TArgParser.Create;
  try
    Args.AddString('c', 'config', 'myapp.ini', 'Config file path');
    Args.AddInteger('p', 'port', 8080, 'Port number');
    Args.AddFlag('d', 'debug', 'Enable debug mode');
    Args.AddFlag('h', 'help', 'Show this help');
    
    if Args.Parse then
    begin
      if Args.GetFlag('help') then
      begin
        Args.PrintHelp;
        Halt(0);
      end;
      
      var ConfigFile := Args.GetString('config', 'myapp.ini');
      var Port := Args.GetInteger('port', 8080);
      var IsDebug := Args.GetFlag('debug');
      
      WriteLn(Format('Config: %s, Port: %d, Debug: %s',
        [ConfigFile, Port, IfThen(IsDebug, 'yes', 'no')]));
    end
    else
    begin
      Args.PrintHelp;
      Halt(1);
    end;
  finally
    Args.Free;
  end;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **INI Config** - อ่าน/เขียน configuration จากไฟล์ INI
2. **JSON Config** - Config แบบ hierarchical ด้วย JSON
3. **Environment Variables** - อ่าน config จาก environment
4. **Command-line Parser** - รับ arguments จาก command line
5. **Config Priority** - CLI > ENV > JSON > INI > Defaults

การจัดการ configuration ที่ดีทำให้แอปพลิเคชันปรับตัวได้ง่ายในสภาพแวดล้อมต่างๆ โดยไม่ต้อง compile ใหม่
