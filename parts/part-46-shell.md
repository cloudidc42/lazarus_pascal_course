# Part 46 - Shell Integration ใน Lazarus/Pascal

## บทนำ

Shell Integration ช่วยให้แอปพลิเคชันสามารถโต้ตอบกับระบบปฏิบัติการได้อย่างลึกซึ้ง เช่น การรัน external processes, การจัดการ environment variables, system tray icons และ Windows notifications

---

## 46.1 Running External Processes

### การรัน Process ด้วย TProcess

```pascal
program RunExternalProcess;

{$mode objfpc}{$H+}

uses
  Process, SysUtils, Classes;

// รัน command และ return output
function RunCommand(const Command: string; 
  const Params: array of string): string;
var
  Proc: TProcess;
  Buffer: string;
  ByteBuf: array[0..4095] of byte;
  BytesRead: integer;
begin
  Result := '';
  Proc := TProcess.Create(nil);
  try
    Proc.Executable := Command;
    var P: string;
    for P in Params do
      Proc.Parameters.Add(P);
    
    Proc.Options := Proc.Options + [poUsePipes, poStderrToOutPut];
    Proc.ShowWindow := swoHide;
    Proc.Execute;
    
    Buffer := '';
    repeat
      BytesRead := Proc.Output.Read(ByteBuf, SizeOf(ByteBuf));
      if BytesRead > 0 then
      begin
        SetLength(Buffer, Length(Buffer) + BytesRead);
        Move(ByteBuf[0], Buffer[Length(Buffer) - BytesRead + 1], BytesRead);
      end;
    until BytesRead = 0;
    
    Proc.WaitOnExit;
    Result := Buffer;
  finally
    Proc.Free;
  end;
end;

// รัน command แบบ async
procedure RunAsync(const Command: string; const Params: array of string);
var
  Proc: TProcess;
begin
  Proc := TProcess.Create(nil);
  try
    Proc.Executable := Command;
    var P: string;
    for P in Params do
      Proc.Parameters.Add(P);
    Proc.Options := Proc.Options - [poWaitOnExit]; // ไม่รอ
    Proc.Execute;
    WriteLn('Process started with PID: ', Proc.ProcessID);
  finally
    Proc.Free;
  end;
end;

// รัน shell command
function RunShell(const Command: string): string;
begin
  {$IFDEF WINDOWS}
  Result := RunCommand('cmd.exe', ['/c', Command]);
  {$ELSE}
  Result := RunCommand('/bin/bash', ['-c', Command]);
  {$ENDIF}
end;

begin
  WriteLn('=== Running External Processes ===');
  WriteLn('');
  
  // แสดงข้อมูลระบบ
  {$IFDEF WINDOWS}
  WriteLn('Windows version:');
  WriteLn(Trim(RunShell('ver')));
  WriteLn('');
  WriteLn('Directory listing:');
  WriteLn(Trim(RunShell('dir /b C:\')));
  {$ELSE}
  WriteLn('OS info:');
  WriteLn(Trim(RunShell('uname -a')));
  WriteLn('');
  WriteLn('Current directory:');
  WriteLn(Trim(RunShell('ls -la')));
  {$ENDIF}
end.
```

---

## 46.2 TProcess

### TProcess Options และ Features

```pascal
program TProcessAdvanced;

{$mode objfpc}{$H+}

uses
  Process, SysUtils, Classes;

type
  TProcessRunner = class
  public
    class function Run(
      const Executable: string;
      const Args: array of string;
      const WorkDir: string = '';
      TimeoutMs: cardinal = 30000
    ): record
      ExitCode: integer;
      Output: string;
      Error: string;
      TimedOut: boolean;
    end;
    
    class function RunPipeline(
      const Commands: array of string
    ): string;
  end;

class function TProcessRunner.Run(
  const Executable: string;
  const Args: array of string;
  const WorkDir: string;
  TimeoutMs: cardinal
): record
  ExitCode: integer;
  Output: string;
  Error: string;
  TimedOut: boolean;
end;
var
  Proc: TProcess;
  OutBuf, ErrBuf: string;
  Bytes: array[0..4095] of byte;
  BytesRead: integer;
  StartTime: TDateTime;
begin
  Result.ExitCode := -1;
  Result.Output := '';
  Result.Error := '';
  Result.TimedOut := False;
  
  Proc := TProcess.Create(nil);
  try
    Proc.Executable := Executable;
    var Arg: string;
    for Arg in Args do
      Proc.Parameters.Add(Arg);
    
    if WorkDir <> '' then
      Proc.CurrentDirectory := WorkDir;
    
    Proc.Options := Proc.Options + [poUsePipes, poNoConsole];
    Proc.ShowWindow := swoHide;
    
    StartTime := Now;
    Proc.Execute;
    
    OutBuf := '';
    ErrBuf := '';
    
    // อ่าน output แบบ non-blocking
    while Proc.Running do
    begin
      // ตรวจสอบ timeout
      if (Now - StartTime) * 86400000 > TimeoutMs then
      begin
        Result.TimedOut := True;
        Proc.Terminate(1);
        Break;
      end;
      
      // อ่าน stdout
      while Proc.Output.NumBytesAvailable > 0 do
      begin
        BytesRead := Proc.Output.Read(Bytes, SizeOf(Bytes));
        if BytesRead > 0 then
        begin
          SetLength(OutBuf, Length(OutBuf) + BytesRead);
          Move(Bytes[0], OutBuf[Length(OutBuf) - BytesRead + 1], BytesRead);
        end;
      end;
      
      // อ่าน stderr
      if poStderrToOutPut in Proc.Options then // ถ้าไม่ได้ redirect
      begin
        while Proc.Stderr.NumBytesAvailable > 0 do
        begin
          BytesRead := Proc.Stderr.Read(Bytes, SizeOf(Bytes));
          if BytesRead > 0 then
          begin
            SetLength(ErrBuf, Length(ErrBuf) + BytesRead);
            Move(Bytes[0], ErrBuf[Length(ErrBuf) - BytesRead + 1], BytesRead);
          end;
        end;
      end;
      
      Sleep(10);
    end;
    
    // อ่านที่เหลือ
    repeat
      BytesRead := Proc.Output.Read(Bytes, SizeOf(Bytes));
      if BytesRead > 0 then
      begin
        SetLength(OutBuf, Length(OutBuf) + BytesRead);
        Move(Bytes[0], OutBuf[Length(OutBuf) - BytesRead + 1], BytesRead);
      end;
    until BytesRead = 0;
    
    Result.ExitCode := Proc.ExitCode;
    Result.Output := OutBuf;
    Result.Error := ErrBuf;
    
  finally
    Proc.Free;
  end;
end;

class function TProcessRunner.RunPipeline(const Commands: array of string): string;
var
  i: integer;
  CurrentInput: string;
  Proc: TProcess;
  Bytes: array[0..4095] of byte;
  BytesRead: integer;
  OutputBuf: string;
  InputStream: TStringStream;
begin
  Result := '';
  CurrentInput := '';
  
  for i := 0 to High(Commands) do
  begin
    Proc := TProcess.Create(nil);
    OutputBuf := '';
    
    try
      {$IFDEF WINDOWS}
      Proc.Executable := 'cmd.exe';
      Proc.Parameters.Add('/c');
      Proc.Parameters.Add(Commands[i]);
      {$ELSE}
      Proc.Executable := '/bin/bash';
      Proc.Parameters.Add('-c');
      Proc.Parameters.Add(Commands[i]);
      {$ENDIF}
      
      Proc.Options := Proc.Options + [poUsePipes, poNoConsole];
      
      if CurrentInput <> '' then
      begin
        InputStream := TStringStream.Create(CurrentInput);
        Proc.Execute;
        
        // ส่ง input
        Proc.Input.Write(InputStream.Memory^, InputStream.Size);
        Proc.CloseInput;
        InputStream.Free;
      end
      else
        Proc.Execute;
      
      // อ่าน output
      Proc.WaitOnExit;
      repeat
        BytesRead := Proc.Output.Read(Bytes, SizeOf(Bytes));
        if BytesRead > 0 then
        begin
          SetLength(OutputBuf, Length(OutputBuf) + BytesRead);
          Move(Bytes[0], OutputBuf[Length(OutputBuf) - BytesRead + 1], BytesRead);
        end;
      until BytesRead = 0;
      
      CurrentInput := OutputBuf;
      
    finally
      Proc.Free;
    end;
  end;
  
  Result := CurrentInput;
end;

var
  R: record
    ExitCode: integer;
    Output: string;
    Error: string;
    TimedOut: boolean;
  end;
begin
  WriteLn('=== Advanced Process Running ===');
  WriteLn('');
  
  {$IFDEF UNIX}
  // Test basic run
  R := TProcessRunner.Run('ls', ['-la', '/tmp']);
  WriteLn('ls /tmp exit code: ', R.ExitCode);
  WriteLn('Output lines: ', Length(R.Output.Split([#10])));
  WriteLn('');
  
  // Test timeout
  WriteLn('Testing timeout...');
  R := TProcessRunner.Run('sleep', ['5'], '', 1000); // 1 second timeout
  if R.TimedOut then
    WriteLn('Process timed out (expected)');
  WriteLn('');
  
  // Test pipeline
  WriteLn('Pipeline: ls | grep .md | wc -l');
  var PipeResult := TProcessRunner.RunPipeline(['ls', 'grep .md']);
  WriteLn('Result: ', Trim(PipeResult));
  {$ELSE}
  WriteLn('Running echo hello:');
  R := TProcessRunner.Run('cmd.exe', ['/c', 'echo hello world']);
  WriteLn('Output: ', Trim(R.Output));
  WriteLn('Exit code: ', R.ExitCode);
  {$ENDIF}
end.
```

---

## 46.3 Command Line Arguments

### การจัดการ Command Line Arguments

```pascal
program CommandLineArgs;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TArgType = (atFlag, atString, atInteger, atFloat);
  
  TArgDef = record
    ShortName: string;
    LongName: string;
    ArgType: TArgType;
    Description: string;
    Required: boolean;
    DefaultValue: string;
  end;
  
  TArgumentParser = class
  private
    FDefs: array of TArgDef;
    FValues: TStringList;
    FFlags: TStringList;
    FPositional: TStringList;
    FProgramName: string;
    FDescription: string;
    
    function FindDef(const Name: string): integer;
    function ValidateValue(const Def: TArgDef; const Value: string): boolean;
  public
    constructor Create(const ProgramName, Description: string);
    destructor Destroy; override;
    
    procedure AddArgument(const ShortName, LongName: string; 
      ArgType: TArgType; const Description: string;
      Required: boolean = False; const Default: string = '');
    
    procedure Parse;
    procedure PrintHelp;
    
    function HasFlag(const Name: string): boolean;
    function GetString(const Name, Default: string): string;
    function GetInteger(const Name: string; Default: integer): integer;
    function GetFloat(const Name: string; Default: double): double;
    function GetPositional(Index: integer): string;
    function GetPositionalCount: integer;
  end;

constructor TArgumentParser.Create(const ProgramName, Description: string);
begin
  inherited Create;
  FProgramName := ProgramName;
  FDescription := Description;
  SetLength(FDefs, 0);
  FValues := TStringList.Create;
  FFlags := TStringList.Create;
  FPositional := TStringList.Create;
end;

destructor TArgumentParser.Destroy;
begin
  FValues.Free;
  FFlags.Free;
  FPositional.Free;
  inherited Destroy;
end;

procedure TArgumentParser.AddArgument(const ShortName, LongName: string;
  ArgType: TArgType; const Description: string; Required: boolean; const Default: string);
var
  Def: TArgDef;
  Len: integer;
begin
  Def.ShortName := ShortName;
  Def.LongName := LongName;
  Def.ArgType := ArgType;
  Def.Description := Description;
  Def.Required := Required;
  Def.DefaultValue := Default;
  
  Len := Length(FDefs);
  SetLength(FDefs, Len + 1);
  FDefs[Len] := Def;
  
  // Set default
  if Default <> '' then
    FValues.Values[LongName] := Default;
end;

function TArgumentParser.FindDef(const Name: string): integer;
var
  i: integer;
  CleanName: string;
begin
  Result := -1;
  CleanName := Name;
  while (CleanName <> '') and (CleanName[1] = '-') do
    Delete(CleanName, 1, 1);
  
  for i := 0 to High(FDefs) do
    if (FDefs[i].ShortName = CleanName) or (FDefs[i].LongName = CleanName) then
    begin
      Result := i;
      Break;
    end;
end;

function TArgumentParser.ValidateValue(const Def: TArgDef; const Value: string): boolean;
var
  IntVal: integer;
  FloatVal: double;
  Code: integer;
begin
  Result := True;
  case Def.ArgType of
    atInteger:
    begin
      Val(Value, IntVal, Code);
      Result := Code = 0;
    end;
    atFloat:
    begin
      Val(Value, FloatVal, Code);
      Result := Code = 0;
    end;
  end;
end;

procedure TArgumentParser.Parse;
var
  i: integer;
  Arg, NextArg: string;
  DefIdx: integer;
begin
  i := 1;
  while i <= ParamCount do
  begin
    Arg := ParamStr(i);
    
    if (Length(Arg) > 0) and (Arg[1] = '-') then
    begin
      DefIdx := FindDef(Arg);
      
      if DefIdx < 0 then
      begin
        WriteLn('Unknown argument: ', Arg);
        Inc(i);
        Continue;
      end;
      
      if FDefs[DefIdx].ArgType = atFlag then
      begin
        FFlags.Add(FDefs[DefIdx].LongName);
      end
      else
      begin
        // ต้องการ value
        if i < ParamCount then
        begin
          Inc(i);
          NextArg := ParamStr(i);
          
          if ValidateValue(FDefs[DefIdx], NextArg) then
            FValues.Values[FDefs[DefIdx].LongName] := NextArg
          else
            WriteLn('Invalid value for ', Arg, ': ', NextArg);
        end
        else
          WriteLn('Missing value for: ', Arg);
      end;
    end
    else
      FPositional.Add(Arg);
    
    Inc(i);
  end;
  
  // ตรวจสอบ required arguments
  for var Def in FDefs do
    if Def.Required and (FValues.IndexOfName(Def.LongName) < 0) then
      WriteLn('Required argument missing: --', Def.LongName);
end;

procedure TArgumentParser.PrintHelp;
var
  Def: TArgDef;
begin
  WriteLn('Usage: ', FProgramName, ' [options] [arguments]');
  WriteLn('');
  WriteLn(FDescription);
  WriteLn('');
  WriteLn('Options:');
  
  for Def in FDefs do
  begin
    var Line: string;
    if Def.ShortName <> '' then
      Line := Format('  -%s, --%-20s', [Def.ShortName, Def.LongName])
    else
      Line := Format('      --%-20s', [Def.LongName]);
      
    Line := Line + Def.Description;
    
    if Def.DefaultValue <> '' then
      Line := Line + Format(' (default: %s)', [Def.DefaultValue]);
    if Def.Required then
      Line := Line + ' [REQUIRED]';
      
    WriteLn(Line);
  end;
  
  WriteLn('');
  WriteLn('  -h, --help               Show this help message');
end;

function TArgumentParser.HasFlag(const Name: string): boolean;
begin
  Result := FFlags.IndexOf(Name) >= 0;
end;

function TArgumentParser.GetString(const Name, Default: string): string;
var
  Idx: integer;
begin
  Idx := FValues.IndexOfName(Name);
  if Idx >= 0 then
    Result := FValues.ValueFromIndex[Idx]
  else
    Result := Default;
end;

function TArgumentParser.GetInteger(const Name: string; Default: integer): integer;
var
  Str: string;
  Code: integer;
begin
  Str := GetString(Name, IntToStr(Default));
  Val(Str, Result, Code);
  if Code <> 0 then
    Result := Default;
end;

function TArgumentParser.GetFloat(const Name: string; Default: double): double;
var
  Str: string;
  Code: integer;
begin
  Str := GetString(Name, FloatToStr(Default));
  Val(Str, Result, Code);
  if Code <> 0 then
    Result := Default;
end;

function TArgumentParser.GetPositional(Index: integer): string;
begin
  if (Index >= 0) and (Index < FPositional.Count) then
    Result := FPositional[Index]
  else
    Result := '';
end;

function TArgumentParser.GetPositionalCount: integer;
begin
  Result := FPositional.Count;
end;

var
  Parser: TArgumentParser;
begin
  Parser := TArgumentParser.Create(
    ExtractFileName(ParamStr(0)),
    'ตัวอย่างโปรแกรมที่ใช้ command line arguments'
  );
  
  try
    Parser.AddArgument('i', 'input', atString, 'Input file path', True);
    Parser.AddArgument('o', 'output', atString, 'Output file path', False, 'output.txt');
    Parser.AddArgument('n', 'max-lines', atInteger, 'Maximum lines to process', False, '1000');
    Parser.AddArgument('v', 'verbose', atFlag, 'Enable verbose output');
    Parser.AddArgument('d', 'debug', atFlag, 'Enable debug mode');
    Parser.AddArgument('', 'format', atString, 'Output format (text/csv/json)', False, 'text');
    
    // ตรวจสอบว่ามี --help
    if (ParamCount = 0) or (ParamStr(1) = '--help') or (ParamStr(1) = '-h') then
    begin
      Parser.PrintHelp;
      Exit;
    end;
    
    Parser.Parse;
    
    WriteLn('=== Parsed Arguments ===');
    WriteLn('Input: ', Parser.GetString('input', ''));
    WriteLn('Output: ', Parser.GetString('output', 'output.txt'));
    WriteLn('Max Lines: ', Parser.GetInteger('max-lines', 1000));
    WriteLn('Format: ', Parser.GetString('format', 'text'));
    WriteLn('Verbose: ', Parser.HasFlag('verbose'));
    WriteLn('Debug: ', Parser.HasFlag('debug'));
    
    if Parser.GetPositionalCount > 0 then
    begin
      WriteLn('');
      WriteLn('Positional arguments:');
      var i: integer;
      for i := 0 to Parser.GetPositionalCount - 1 do
        WriteLn('  [', i, ']: ', Parser.GetPositional(i));
    end;
    
  finally
    Parser.Free;
  end;
end.
```

---

## 46.4 Environment Variables

### การจัดการ Environment Variables

```pascal
program EnvironmentVariables;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TEnvManager = class
  public
    // อ่าน env variable
    class function Get(const Name: string; const Default: string = ''): string;
    // ตรวจสอบว่ามี env variable
    class function Exists(const Name: string): boolean;
    // ตั้งค่า env variable (สำหรับ process ปัจจุบันและ child processes)
    class function SetVar(const Name, Value: string): boolean;
    // ลบ env variable
    class function Unset(const Name: string): boolean;
    // แสดง env variables ทั้งหมด
    class procedure List(Vars: TStringList);
    // Expand env variables ใน string (เช่น %PATH% หรือ $HOME)
    class function Expand(const Value: string): string;
  end;

class function TEnvManager.Get(const Name: string; const Default: string): string;
begin
  Result := GetEnvironmentVariable(Name);
  if Result = '' then
    Result := Default;
end;

class function TEnvManager.Exists(const Name: string): boolean;
begin
  Result := GetEnvironmentVariable(Name) <> '';
end;

class function TEnvManager.SetVar(const Name, Value: string): boolean;
begin
  Result := SetEnvironmentVariable(PChar(Name), PChar(Value));
end;

class function TEnvManager.Unset(const Name: string): boolean;
begin
  Result := SetEnvironmentVariable(PChar(Name), nil);
end;

class procedure TEnvManager.List(Vars: TStringList);
var
  i: integer;
  EnvStr: string;
begin
  Vars.Clear;
  i := 1;
  repeat
    EnvStr := GetEnvironmentString(i);
    if EnvStr <> '' then
      Vars.Add(EnvStr);
    Inc(i);
  until EnvStr = '';
end;

class function TEnvManager.Expand(const Value: string): string;
begin
  {$IFDEF WINDOWS}
  // Windows: ขยาย %VARNAME%
  SetLength(Result, 4096);
  var Len := ExpandEnvironmentStrings(PChar(Value), PChar(Result), 4096);
  SetLength(Result, Len - 1);
  {$ELSE}
  // Unix: ใช้ bash -c echo
  Result := Value;
  // simple expansion ด้วย GetEnvironmentVariable
  var i: integer;
  var CurPos := 1;
  var Output := '';
  
  while CurPos <= Length(Value) do
  begin
    if Value[CurPos] = '$' then
    begin
      Inc(CurPos);
      var VarName := '';
      if (CurPos <= Length(Value)) and (Value[CurPos] = '{') then
      begin
        Inc(CurPos); // skip {
        while (CurPos <= Length(Value)) and (Value[CurPos] <> '}') do
        begin
          VarName := VarName + Value[CurPos];
          Inc(CurPos);
        end;
        Inc(CurPos); // skip }
      end
      else
      begin
        while (CurPos <= Length(Value)) and 
              (Value[CurPos] in ['A'..'Z', 'a'..'z', '0'..'9', '_']) do
        begin
          VarName := VarName + Value[CurPos];
          Inc(CurPos);
        end;
      end;
      
      Output := Output + GetEnvironmentVariable(VarName);
    end
    else
    begin
      Output := Output + Value[CurPos];
      Inc(CurPos);
    end;
  end;
  
  Result := Output;
  {$ENDIF}
end;

procedure DemoEnvVars;
var
  Vars: TStringList;
  VarName: string;
begin
  WriteLn('=== Environment Variables Demo ===');
  WriteLn('');
  
  // อ่าน common env vars
  {$IFDEF WINDOWS}
  WriteLn('PATH: ', Copy(TEnvManager.Get('PATH'), 1, 80), '...');
  WriteLn('USERNAME: ', TEnvManager.Get('USERNAME'));
  WriteLn('COMPUTERNAME: ', TEnvManager.Get('COMPUTERNAME'));
  WriteLn('TEMP: ', TEnvManager.Get('TEMP'));
  WriteLn('PROGRAMFILES: ', TEnvManager.Get('ProgramFiles'));
  {$ELSE}
  WriteLn('PATH: ', Copy(TEnvManager.Get('PATH'), 1, 80), '...');
  WriteLn('USER: ', TEnvManager.Get('USER'));
  WriteLn('HOME: ', TEnvManager.Get('HOME'));
  WriteLn('SHELL: ', TEnvManager.Get('SHELL'));
  WriteLn('LANG: ', TEnvManager.Get('LANG'));
  {$ENDIF}
  
  WriteLn('');
  WriteLn('Missing var with default: ',
    TEnvManager.Get('NON_EXISTENT_VAR', 'default_value'));
  
  // ตั้งค่า env var
  WriteLn('');
  WriteLn('Setting custom env vars...');
  TEnvManager.SetVar('MY_APP_VERSION', '2.0.1');
  TEnvManager.SetVar('MY_APP_ENV', 'production');
  TEnvManager.SetVar('MY_DEBUG', 'false');
  
  WriteLn('MY_APP_VERSION: ', TEnvManager.Get('MY_APP_VERSION'));
  WriteLn('MY_APP_ENV: ', TEnvManager.Get('MY_APP_ENV'));
  
  // Expand
  WriteLn('');
  {$IFDEF WINDOWS}
  WriteLn('Expanded: ', TEnvManager.Expand('%USERPROFILE%\Documents'));
  {$ELSE}
  WriteLn('Expanded: ', TEnvManager.Expand('$HOME/Documents'));
  {$ENDIF}
  
  // ลบ env var
  TEnvManager.Unset('MY_APP_VERSION');
  WriteLn('After unset MY_APP_VERSION: [', TEnvManager.Get('MY_APP_VERSION'), ']');
  
  // แสดง all env vars (แค่ 10 แรก)
  Vars := TStringList.Create;
  try
    TEnvManager.List(Vars);
    WriteLn('');
    WriteLn('Total env variables: ', Vars.Count);
    WriteLn('First 10:');
    var i: integer;
    for i := 0 to Min(9, Vars.Count - 1) do
    begin
      // แสดงเฉพาะ name (ไม่แสดง value ที่อาจมี sensitive info)
      var EqPos := Pos('=', Vars[i]);
      if EqPos > 0 then
        WriteLn('  ', Copy(Vars[i], 1, EqPos - 1));
    end;
  finally
    Vars.Free;
  end;
  
  // ตั้งค่าสำหรับ child processes
  WriteLn('');
  WriteLn('Setting env for child process...');
  TEnvManager.SetVar('GREETING', 'สวัสดีครับ');
  
  {$IFDEF UNIX}
  WriteLn('env | grep GREETING:');
  WriteLn(TEnvManager.Get('GREETING'));
  {$ELSE}
  WriteLn('GREETING = ', TEnvManager.Get('GREETING'));
  {$ENDIF}
end;

begin
  DemoEnvVars;
end.
```

---

## 46.5 System Tray Icons

### System Tray Icon สำหรับ Console/Forms App

```pascal
// system_tray_demo.lpr - สำหรับ Lazarus Forms Application
program SystemTrayDemo;

{$mode objfpc}{$H+}

uses
  Interfaces, Forms, Classes, Controls, Menus, ExtCtrls, SysUtils
  {$IFDEF WINDOWS}, Windows, ShellAPI{$ENDIF};

type
  TMainForm = class(TForm)
  private
    FTrayIcon: TTrayIcon;
    FPopupMenu: TPopupMenu;
    FTimer: TTimer;
    FMinimizeToTray: boolean;
    
    procedure CreateTrayIcon;
    procedure TrayIconClick(Sender: TObject);
    procedure TrayIconDblClick(Sender: TObject);
    procedure MenuShowClick(Sender: TObject);
    procedure MenuAboutClick(Sender: TObject);
    procedure MenuExitClick(Sender: TObject);
    procedure TimerTick(Sender: TObject);
    procedure FormCloseQuery(Sender: TObject; var CanClose: boolean);
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure ShowBalloonHint(const Title, Message: string; 
      BalloonFlags: integer = 0);
    procedure UpdateTrayIcon(const Hint: string);
  end;

constructor TMainForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  Caption := 'System Tray Demo';
  Width := 400;
  Height := 300;
  FMinimizeToTray := True;
  
  CreateTrayIcon;
  
  // Timer สำหรับ update tray hint ทุก 5 วินาที
  FTimer := TTimer.Create(Self);
  FTimer.Interval := 5000;
  FTimer.OnTimer := @TimerTick;
  FTimer.Enabled := True;
end;

destructor TMainForm.Destroy;
begin
  if Assigned(FTrayIcon) then
    FTrayIcon.Visible := False;
  inherited Destroy;
end;

procedure TMainForm.CreateTrayIcon;
var
  MenuShowItem, MenuAboutItem, MenuSepItem, MenuExitItem: TMenuItem;
begin
  // สร้าง popup menu
  FPopupMenu := TPopupMenu.Create(Self);
  
  MenuShowItem := TMenuItem.Create(FPopupMenu);
  MenuShowItem.Caption := 'แสดงหน้าต่าง';
  MenuShowItem.OnClick := @MenuShowClick;
  MenuShowItem.Default := True;
  FPopupMenu.Items.Add(MenuShowItem);
  
  MenuAboutItem := TMenuItem.Create(FPopupMenu);
  MenuAboutItem.Caption := 'เกี่ยวกับ...';
  MenuAboutItem.OnClick := @MenuAboutClick;
  FPopupMenu.Items.Add(MenuAboutItem);
  
  MenuSepItem := TMenuItem.Create(FPopupMenu);
  MenuSepItem.Caption := '-';
  FPopupMenu.Items.Add(MenuSepItem);
  
  MenuExitItem := TMenuItem.Create(FPopupMenu);
  MenuExitItem.Caption := 'ออกจากโปรแกรม';
  MenuExitItem.OnClick := @MenuExitClick;
  FPopupMenu.Items.Add(MenuExitItem);
  
  // สร้าง tray icon
  FTrayIcon := TTrayIcon.Create(Self);
  FTrayIcon.PopupMenu := FPopupMenu;
  FTrayIcon.Hint := 'My Application v1.0';
  FTrayIcon.Visible := True;
  
  // ลองโหลด icon
  try
    // ถ้ามีไฟล์ icon
    // FTrayIcon.Icon.LoadFromFile('app.ico');
    
    // หรือใช้ icon จาก application
    FTrayIcon.Icon.Assign(Application.Icon);
  except
    // ใช้ default icon
  end;
  
  FTrayIcon.OnClick := @TrayIconClick;
  FTrayIcon.OnDblClick := @TrayIconDblClick;
end;

procedure TMainForm.TrayIconClick(Sender: TObject);
begin
  // คลิกซ้ายครั้งเดียว
end;

procedure TMainForm.TrayIconDblClick(Sender: TObject);
begin
  // ดับเบิลคลิก = แสดงหน้าต่าง
  MenuShowClick(Sender);
end;

procedure TMainForm.MenuShowClick(Sender: TObject);
begin
  Show;
  WindowState := wsNormal;
  BringToFront;
  Application.BringToFront;
end;

procedure TMainForm.MenuAboutClick(Sender: TObject);
begin
  ShowMessage('System Tray Demo v1.0'#13#10'เขียนด้วย Lazarus Pascal');
end;

procedure TMainForm.MenuExitClick(Sender: TObject);
begin
  FMinimizeToTray := False;
  Application.Terminate;
end;

procedure TMainForm.TimerTick(Sender: TObject);
begin
  UpdateTrayIcon(Format('เวลา: %s', [TimeToStr(Now)]));
end;

procedure TMainForm.FormCloseQuery(Sender: TObject; var CanClose: boolean);
begin
  if FMinimizeToTray then
  begin
    Hide; // ซ่อนแทนการปิด
    
    ShowBalloonHint(
      'โปรแกรมยังทำงานอยู่',
      'โปรแกรมถูกย่อลง System Tray คลิกที่ไอคอนเพื่อแสดงหน้าต่าง',
      NIIF_INFO
    );
    
    CanClose := False;
  end;
end;

procedure TMainForm.ShowBalloonHint(const Title, Message: string; BalloonFlags: integer);
begin
  FTrayIcon.BalloonTitle := Title;
  FTrayIcon.BalloonHint := Message;
  case BalloonFlags of
    1: FTrayIcon.BalloonFlags := bfInfo;
    2: FTrayIcon.BalloonFlags := bfWarning;
    3: FTrayIcon.BalloonFlags := bfError;
    else FTrayIcon.BalloonFlags := bfNone;
  end;
  FTrayIcon.ShowBalloonHint;
end;

procedure TMainForm.UpdateTrayIcon(const Hint: string);
begin
  FTrayIcon.Hint := Hint;
end;

begin
  Application.Initialize;
  Application.CreateForm(TMainForm, Application.MainForm);
  Application.Run;
end.
```

---

## 46.6 Windows Notifications

### Windows Toast Notifications

```pascal
program WindowsNotifications;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Windows, SysUtils, ComObj, Variants;

type
  TNotificationIcon = (niNone, niInfo, niWarning, niError);

  TWindowsNotifier = class
  public
    // Simple balloon notification ผ่าน Shell
    class procedure ShowBalloon(
      const Title, Message: string;
      Icon: TNotificationIcon = niInfo;
      TimeoutMs: integer = 5000
    );
    
    // Toast Notification (Windows 10+)
    class procedure ShowToast(
      const AppID, Title, Message: string;
      const ImagePath: string = ''
    );
    
    // Message Box notifications
    class function ShowDialog(
      const Title, Message: string;
      DialogType: integer = MB_OK or MB_ICONINFORMATION
    ): integer;
  end;

class procedure TWindowsNotifier.ShowBalloon(
  const Title, Message: string;
  Icon: TNotificationIcon;
  TimeoutMs: integer
);
var
  NotifyData: NOTIFYICONDATA;
  hwnd: HWND;
begin
  hwnd := CreateWindowEx(0, 'STATIC', nil, 0, 0, 0, 0, 0, 
    HWND_MESSAGE, 0, 0, nil);
  
  FillChar(NotifyData, SizeOf(NotifyData), 0);
  NotifyData.cbSize := SizeOf(NOTIFYICONDATA);
  NotifyData.Wnd := hwnd;
  NotifyData.uID := 1;
  NotifyData.uFlags := NIF_INFO;
  NotifyData.uTimeout := TimeoutMs;
  
  StrLCopy(NotifyData.szInfoTitle, PChar(Title), 64);
  StrLCopy(NotifyData.szInfo, PChar(Message), 256);
  
  case Icon of
    niInfo: NotifyData.dwInfoFlags := NIIF_INFO;
    niWarning: NotifyData.dwInfoFlags := NIIF_WARNING;
    niError: NotifyData.dwInfoFlags := NIIF_ERROR;
    else NotifyData.dwInfoFlags := NIIF_NONE;
  end;
  
  NotifyData.uFlags := NotifyData.uFlags or NIF_ICON;
  NotifyData.hIcon := LoadIcon(0, IDI_APPLICATION);
  
  // เพิ่ม icon ก่อน
  Shell_NotifyIcon(NIM_ADD, @NotifyData);
  
  // แก้ไข (จะแสดง balloon)
  Shell_NotifyIcon(NIM_MODIFY, @NotifyData);
  
  // รอสักครู่แล้วลบ icon
  Sleep(TimeoutMs + 1000);
  Shell_NotifyIcon(NIM_DELETE, @NotifyData);
  DestroyWindow(hwnd);
end;

class procedure TWindowsNotifier.ShowToast(
  const AppID, Title, Message: string;
  const ImagePath: string
);
begin
  // Windows 10+ Toast Notifications ต้องใช้ WinRT API
  // ซึ่งต้องการ COM/WinRT interop
  // สำหรับการใช้งานจริงควรใช้ library เช่น WinToast
  WriteLn('Toast Notification:');
  WriteLn('AppID: ', AppID);
  WriteLn('Title: ', Title);
  WriteLn('Message: ', Message);
  if ImagePath <> '' then
    WriteLn('Image: ', ImagePath);
end;

class function TWindowsNotifier.ShowDialog(
  const Title, Message: string;
  DialogType: integer
): integer;
begin
  Result := MessageBox(0, PChar(Message), PChar(Title), DialogType);
end;

begin
  WriteLn('=== Windows Notifications Demo ===');
  WriteLn('');
  
  // Message Box
  WriteLn('Showing message box...');
  var Result := TWindowsNotifier.ShowDialog(
    'การแจ้งเตือน',
    'นี่คือการแจ้งเตือนแบบ message box' + #13#10 + 'คลิก OK เพื่อดำเนินการต่อ',
    MB_OK or MB_ICONINFORMATION
  );
  
  WriteLn('User clicked: ', case Result of
    IDOK: 'OK';
    IDCANCEL: 'Cancel';
    IDYES: 'Yes';
    IDNO: 'No';
    else 'Unknown';
  end);
  
  WriteLn('');
  WriteLn('Showing balloon notification...');
  TWindowsNotifier.ShowBalloon(
    'แจ้งเตือน',
    'นี่คือ balloon notification จาก Lazarus Pascal',
    niInfo,
    3000
  );
  
  WriteLn('Notification demo completed!');
end.

{$ELSE}
uses
  SysUtils;
begin
  WriteLn('Windows Notifications demo is Windows-only');
  WriteLn('On Linux, use libnotify or dbus notifications');
  WriteLn('On macOS, use NSUserNotification or UNUserNotificationCenter');
end.
{$ENDIF}
```

---

## 46.7 Complete Example: File Manager

```pascal
program SimpleFileManager;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes, Process;

type
  TFileInfo = record
    Name: string;
    Size: int64;
    Modified: TDateTime;
    IsDir: boolean;
    Attributes: integer;
    Extension: string;
  end;
  
  TFileList = array of TFileInfo;

  TSimpleFileManager = class
  private
    FCurrentPath: string;
    FHistory: TStringList;
    FHistoryPos: integer;
    
    function FormatSize(Bytes: int64): string;
    function FormatTime(DT: TDateTime): string;
  public
    constructor Create(const StartPath: string = '');
    destructor Destroy; override;
    
    function Navigate(const Path: string): boolean;
    function GoUp: boolean;
    function GoBack: boolean;
    function GoForward: boolean;
    
    function ListFiles(const Filter: string = '*'): TFileList;
    function GetFileInfo(const FileName: string): TFileInfo;
    
    procedure CopyFile(const Source, Dest: string);
    procedure MoveFile(const Source, Dest: string);
    procedure DeleteFile(const FileName: string);
    procedure CreateDirectory(const DirName: string);
    
    procedure OpenFile(const FileName: string);
    procedure OpenInExplorer;
    
    procedure Display(const Filter: string = '*');
    procedure DisplayStats;
    
    property CurrentPath: string read FCurrentPath;
  end;

constructor TSimpleFileManager.Create(const StartPath: string);
begin
  inherited Create;
  FHistory := TStringList.Create;
  FHistoryPos := -1;
  
  if StartPath <> '' then
    Navigate(StartPath)
  else
    Navigate(GetCurrentDir);
end;

destructor TSimpleFileManager.Destroy;
begin
  FHistory.Free;
  inherited Destroy;
end;

function TSimpleFileManager.FormatSize(Bytes: int64): string;
const
  Units: array[0..4] of string = ('B', 'KB', 'MB', 'GB', 'TB');
var
  Size: double;
  UnitIdx: integer;
begin
  Size := Bytes;
  UnitIdx := 0;
  
  while (Size >= 1024) and (UnitIdx < 4) do
  begin
    Size := Size / 1024;
    Inc(UnitIdx);
  end;
  
  if UnitIdx = 0 then
    Result := Format('%d %s', [Bytes, Units[0]])
  else
    Result := Format('%.2f %s', [Size, Units[UnitIdx]]);
end;

function TSimpleFileManager.FormatTime(DT: TDateTime): string;
begin
  Result := FormatDateTime('yyyy-mm-dd hh:nn', DT);
end;

function TSimpleFileManager.Navigate(const Path: string): boolean;
var
  AbsPath: string;
begin
  AbsPath := ExpandFileName(Path);
  
  if not DirectoryExists(AbsPath) then
    Exit(False);
  
  // บันทึก history
  if (FHistoryPos < 0) or (FHistory[FHistoryPos] <> AbsPath) then
  begin
    // ลบ forward history
    while FHistory.Count > FHistoryPos + 1 do
      FHistory.Delete(FHistory.Count - 1);
    
    FHistory.Add(AbsPath);
    FHistoryPos := FHistory.Count - 1;
  end;
  
  FCurrentPath := AbsPath;
  Result := True;
end;

function TSimpleFileManager.GoUp: boolean;
begin
  var Parent := ExtractFilePath(ExcludeTrailingPathDelimiter(FCurrentPath));
  if Parent = FCurrentPath then
    Exit(False);
  Result := Navigate(Parent);
end;

function TSimpleFileManager.GoBack: boolean;
begin
  if FHistoryPos <= 0 then
    Exit(False);
  Dec(FHistoryPos);
  FCurrentPath := FHistory[FHistoryPos];
  Result := True;
end;

function TSimpleFileManager.GoForward: boolean;
begin
  if FHistoryPos >= FHistory.Count - 1 then
    Exit(False);
  Inc(FHistoryPos);
  FCurrentPath := FHistory[FHistoryPos];
  Result := True;
end;

function TSimpleFileManager.ListFiles(const Filter: string): TFileList;
var
  SR: TSearchRec;
  Count: integer;
begin
  Count := 0;
  SetLength(Result, 0);
  
  // แสดง directories ก่อน
  if FindFirst(IncludeTrailingPathDelimiter(FCurrentPath) + '*', 
    faDirectory, SR) = 0 then
  begin
    repeat
      if (SR.Name = '.') or (SR.Name = '..') then Continue;
      if (SR.Attr and faDirectory) = 0 then Continue;
      
      SetLength(Result, Count + 1);
      Result[Count].Name := SR.Name;
      Result[Count].IsDir := True;
      Result[Count].Size := 0;
      Result[Count].Modified := SR.TimeStamp;
      Result[Count].Attributes := SR.Attr;
      Result[Count].Extension := '';
      Inc(Count);
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
  
  // แสดง files
  if FindFirst(IncludeTrailingPathDelimiter(FCurrentPath) + Filter, 
    faAnyFile - faDirectory, SR) = 0 then
  begin
    repeat
      if (SR.Attr and faDirectory) <> 0 then Continue;
      
      SetLength(Result, Count + 1);
      Result[Count].Name := SR.Name;
      Result[Count].IsDir := False;
      Result[Count].Size := SR.Size;
      Result[Count].Modified := SR.TimeStamp;
      Result[Count].Attributes := SR.Attr;
      Result[Count].Extension := LowerCase(ExtractFileExt(SR.Name));
      Inc(Count);
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
end;

function TSimpleFileManager.GetFileInfo(const FileName: string): TFileInfo;
var
  SR: TSearchRec;
  FullPath: string;
begin
  FillChar(Result, SizeOf(Result), 0);
  FullPath := IncludeTrailingPathDelimiter(FCurrentPath) + FileName;
  
  if FindFirst(FullPath, faAnyFile, SR) = 0 then
  begin
    Result.Name := SR.Name;
    Result.Size := SR.Size;
    Result.Modified := SR.TimeStamp;
    Result.Attributes := SR.Attr;
    Result.IsDir := (SR.Attr and faDirectory) <> 0;
    Result.Extension := LowerCase(ExtractFileExt(SR.Name));
    FindClose(SR);
  end;
end;

procedure TSimpleFileManager.CopyFile(const Source, Dest: string);
var
  SrcPath, DstPath: string;
begin
  SrcPath := IncludeTrailingPathDelimiter(FCurrentPath) + Source;
  DstPath := IncludeTrailingPathDelimiter(FCurrentPath) + Dest;
  
  if not FileExists(SrcPath) then
  begin
    WriteLn('Source not found: ', Source);
    Exit;
  end;
  
  if not SysUtils.CopyFile(SrcPath, DstPath, True) then
    WriteLn('Copy failed!')
  else
    WriteLn('Copied: ', Source, ' -> ', Dest);
end;

procedure TSimpleFileManager.MoveFile(const Source, Dest: string);
var
  SrcPath, DstPath: string;
begin
  SrcPath := IncludeTrailingPathDelimiter(FCurrentPath) + Source;
  DstPath := IncludeTrailingPathDelimiter(FCurrentPath) + Dest;
  
  if not RenameFile(SrcPath, DstPath) then
    WriteLn('Move failed!')
  else
    WriteLn('Moved: ', Source, ' -> ', Dest);
end;

procedure TSimpleFileManager.DeleteFile(const FileName: string);
var
  FullPath: string;
begin
  FullPath := IncludeTrailingPathDelimiter(FCurrentPath) + FileName;
  
  if DirectoryExists(FullPath) then
  begin
    if not RemoveDir(FullPath) then
      WriteLn('Cannot delete directory (not empty?)')
    else
      WriteLn('Deleted directory: ', FileName);
  end
  else if FileExists(FullPath) then
  begin
    if not SysUtils.DeleteFile(FullPath) then
      WriteLn('Cannot delete file')
    else
      WriteLn('Deleted file: ', FileName);
  end
  else
    WriteLn('Not found: ', FileName);
end;

procedure TSimpleFileManager.CreateDirectory(const DirName: string);
var
  FullPath: string;
begin
  FullPath := IncludeTrailingPathDelimiter(FCurrentPath) + DirName;
  
  if MkDir(FullPath) then
    WriteLn('Created directory: ', DirName)
  else
    WriteLn('Failed to create directory');
end;

procedure TSimpleFileManager.OpenFile(const FileName: string);
var
  FullPath: string;
begin
  FullPath := IncludeTrailingPathDelimiter(FCurrentPath) + FileName;
  
  {$IFDEF WINDOWS}
  ShellExecute(0, 'open', PChar(FullPath), nil, nil, SW_SHOW);
  {$ENDIF}
  {$IFDEF LINUX}
  var Proc := TProcess.Create(nil);
  try
    Proc.Executable := 'xdg-open';
    Proc.Parameters.Add(FullPath);
    Proc.Execute;
  finally
    Proc.Free;
  end;
  {$ENDIF}
  {$IFDEF DARWIN}
  var Proc := TProcess.Create(nil);
  try
    Proc.Executable := 'open';
    Proc.Parameters.Add(FullPath);
    Proc.Execute;
  finally
    Proc.Free;
  end;
  {$ENDIF}
end;

procedure TSimpleFileManager.OpenInExplorer;
begin
  {$IFDEF WINDOWS}
  ShellExecute(0, 'explore', PChar(FCurrentPath), nil, nil, SW_SHOW);
  {$ENDIF}
  {$IFDEF LINUX}
  var Proc := TProcess.Create(nil);
  try
    Proc.Executable := 'xdg-open';
    Proc.Parameters.Add(FCurrentPath);
    Proc.Execute;
  finally
    Proc.Free;
  end;
  {$ENDIF}
  {$IFDEF DARWIN}
  var Proc := TProcess.Create(nil);
  try
    Proc.Executable := 'open';
    Proc.Parameters.Add(FCurrentPath);
    Proc.Execute;
  finally
    Proc.Free;
  end;
  {$ENDIF}
end;

procedure TSimpleFileManager.Display(const Filter: string);
var
  Files: TFileList;
  F: TFileInfo;
begin
  Files := ListFiles(Filter);
  
  WriteLn('');
  WriteLn('Directory: ', FCurrentPath);
  WriteLn('');
  WriteLn(Format('%-5s %-30s %12s %-20s', ['Type', 'Name', 'Size', 'Modified']));
  WriteLn(StringOfChar('-', 75));
  
  for F in Files do
  begin
    var TypeStr: string;
    var SizeStr: string;
    
    if F.IsDir then
    begin
      TypeStr := '[DIR]';
      SizeStr := '<DIR>';
    end
    else
    begin
      TypeStr := '[FIL]';
      SizeStr := FormatSize(F.Size);
    end;
    
    WriteLn(Format('%-5s %-30s %12s %-20s', [
      TypeStr,
      Copy(F.Name, 1, 30),
      SizeStr,
      FormatTime(F.Modified)
    ]));
  end;
  
  WriteLn('');
  WriteLn(Length(Files), ' items');
end;

procedure TSimpleFileManager.DisplayStats;
var
  Files: TFileList;
  F: TFileInfo;
  TotalSize: int64;
  FileCount, DirCount: integer;
begin
  Files := ListFiles('*');
  TotalSize := 0;
  FileCount := 0;
  DirCount := 0;
  
  for F in Files do
    if F.IsDir then
      Inc(DirCount)
    else
    begin
      Inc(FileCount);
      TotalSize := TotalSize + F.Size;
    end;
  
  WriteLn('=== Directory Statistics ===');
  WriteLn('Path: ', FCurrentPath);
  WriteLn('Files: ', FileCount);
  WriteLn('Directories: ', DirCount);
  WriteLn('Total size: ', FormatSize(TotalSize));
end;

var
  FM: TSimpleFileManager;
  TestDir: string;
begin
  WriteLn('=== Simple File Manager Demo ===');
  
  // เริ่มที่ temp directory
  FM := TSimpleFileManager.Create(GetTempDir);
  try
    FM.Display('*');
    FM.DisplayStats;
    
    WriteLn('');
    WriteLn('Creating test directory...');
    TestDir := 'fm_test_' + IntToStr(GetTickCount64 mod 10000);
    FM.CreateDirectory(TestDir);
    
    WriteLn('');
    FM.Display(TestDir + '*');
    
    // ลบ test directory
    FM.DeleteFile(TestDir);
    
  finally
    FM.Free;
  end;
end.
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: Script Executor
สร้าง script executor ที่:
- รับ script file เป็น input
- Execute ทีละ line
- Capture output และ errors
- รองรับ timeout

### ข้อ 2: Environment Manager
สร้าง GUI tool จัดการ environment variables:
- แสดง all env variables
- เพิ่ม/แก้ไข/ลบ variables
- Save/Load preset configurations

### ข้อ 3: Directory Watcher
สร้าง directory watcher service:
- ติดตาม file changes ใน directory
- แจ้งเตือนเมื่อมี new/modified/deleted files
- Log การเปลี่ยนแปลง

### ข้อ 4: Drag and Drop Handler
สร้าง form ที่รองรับ drag and drop:
- รับ files ที่ถูก drop
- แสดง file info
- Process files ที่ได้รับ

### ข้อ 5: System Info Collector
สร้าง system information collector:
- CPU usage
- Memory usage
- Disk space
- Running processes
- แสดงใน dashboard UI
