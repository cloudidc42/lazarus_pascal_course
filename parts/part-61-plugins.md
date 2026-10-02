# ตอนที่ 61: Plugin Architecture ใน Lazarus/Pascal

## บทนำ

Plugin Architecture คือการออกแบบโปรแกรมให้สามารถโหลดโมดูลเพิ่มเติมได้ในขณะรันไทม์ โดยไม่ต้องคอมไพล์โปรแกรมหลักใหม่ ทำให้ระบบมีความยืดหยุ่นและสามารถขยายฟังก์ชันได้ง่าย

### ประโยชน์ของ Plugin Architecture
- ขยายระบบได้โดยไม่ต้องแก้ไขโปรแกรมหลัก
- โหลด/ปลด Plugin ได้ขณะรันไทม์ (Hot-reload)
- แบ่งทีมพัฒนาได้อิสระ
- ลดขนาดโปรแกรมหลัก

---

## 61.1 โครงสร้างพื้นฐาน Plugin

### Interface ที่ Plugin ต้องใช้

สร้างไฟล์ `plugin_interface.pas` สำหรับกำหนด Interface มาตรฐาน:

```pascal
unit plugin_interface;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

const
  PLUGIN_API_VERSION = 1;

type
  // ข้อมูล Plugin
  TPluginInfo = record
    Name: string;
    Version: string;
    Author: string;
    Description: string;
    APIVersion: Integer;
  end;

  // Interface หลักที่ทุก Plugin ต้องใช้
  IPlugin = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    function GetInfo: TPluginInfo;
    function Initialize(AHost: TObject): Boolean;
    procedure Finalize;
    function Execute(const ACommand: string; AParams: TStringList): string;
  end;

  // Interface สำหรับ Plugin ที่มี UI
  IPluginUI = interface(IPlugin)
    ['{B2C3D4E5-F6A7-8901-BCDE-F12345678901}']
    procedure ShowWindow;
    procedure HideWindow;
    function IsVisible: Boolean;
  end;

  // ฟังก์ชันที่ DLL/SO ต้องส่งออก
  TGetPluginInfo = function: TPluginInfo; stdcall;
  TCreatePlugin = function: IPlugin; stdcall;
  TDestroyPlugin = procedure(APlugin: IPlugin); stdcall;

implementation

end.
```

---

## 61.2 สร้าง Plugin (DLL/Shared Library)

### ตัวอย่าง Plugin คำนวณตัวเลข (`calc_plugin.pas`)

```pascal
library calc_plugin;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, Math,
  plugin_interface;

type
  TCalcPlugin = class(TInterfacedObject, IPlugin)
  private
    FHost: TObject;
    FLog: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    
    // IPlugin implementation
    function GetInfo: TPluginInfo;
    function Initialize(AHost: TObject): Boolean;
    procedure Finalize;
    function Execute(const ACommand: string; AParams: TStringList): string;
    
    // ฟังก์ชันภายใน
    function DoCalculate(const AExpr: string): string;
    function DoStatistics(AParams: TStringList): string;
  end;

{ TCalcPlugin }

constructor TCalcPlugin.Create;
begin
  inherited Create;
  FLog := TStringList.Create;
  FLog.Add(FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + ' - CalcPlugin created');
end;

destructor TCalcPlugin.Destroy;
begin
  FLog.Free;
  inherited Destroy;
end;

function TCalcPlugin.GetInfo: TPluginInfo;
begin
  Result.Name := 'Calculator Plugin';
  Result.Version := '1.0.0';
  Result.Author := 'Dev Team';
  Result.Description := 'Plugin สำหรับคำนวณทางคณิตศาสตร์';
  Result.APIVersion := PLUGIN_API_VERSION;
end;

function TCalcPlugin.Initialize(AHost: TObject): Boolean;
begin
  FHost := AHost;
  FLog.Add(FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + ' - Initialized');
  Result := True;
end;

procedure TCalcPlugin.Finalize;
begin
  FLog.SaveToFile('calc_plugin.log');
  FLog.Add(FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + ' - Finalized');
end;

function TCalcPlugin.Execute(const ACommand: string; AParams: TStringList): string;
begin
  Result := '';
  
  if ACommand = 'calculate' then
  begin
    if AParams.Count > 0 then
      Result := DoCalculate(AParams[0])
    else
      Result := 'Error: ไม่มีนิพจน์';
  end
  else if ACommand = 'stats' then
    Result := DoStatistics(AParams)
  else if ACommand = 'version' then
    Result := GetInfo.Version
  else if ACommand = 'help' then
    Result := 'Commands: calculate, stats, version, help'
  else
    Result := 'Error: ไม่รู้จักคำสั่ง ' + ACommand;
    
  FLog.Add(FormatDateTime('hh:nn:ss', Now) + ' - Execute: ' + ACommand + ' = ' + Result);
end;

function TCalcPlugin.DoCalculate(const AExpr: string): string;
var
  Val1, Val2, Answer: Double;
  Op: Char;
  Parts: TStringList;
begin
  // แยกนิพจน์อย่างง่าย: "10 + 5", "20 * 3" ฯลฯ
  Parts := TStringList.Create;
  try
    Parts.Delimiter := ' ';
    Parts.DelimitedText := AExpr;
    
    if Parts.Count >= 3 then
    begin
      Val1 := StrToFloatDef(Parts[0], 0);
      Op := Parts[1][1];
      Val2 := StrToFloatDef(Parts[2], 0);
      
      case Op of
        '+': Answer := Val1 + Val2;
        '-': Answer := Val1 - Val2;
        '*': Answer := Val1 * Val2;
        '/': 
          begin
            if Val2 = 0 then
            begin
              Result := 'Error: หารด้วยศูนย์ไม่ได้';
              Exit;
            end;
            Answer := Val1 / Val2;
          end;
        '^': Answer := Power(Val1, Val2);
        else
        begin
          Result := 'Error: ไม่รู้จักตัวดำเนินการ ' + Op;
          Exit;
        end;
      end;
      
      Result := FloatToStrF(Answer, ffGeneral, 15, 6);
    end
    else
      Result := 'Error: รูปแบบไม่ถูกต้อง (ต้องการ: num op num)';
  finally
    Parts.Free;
  end;
end;

function TCalcPlugin.DoStatistics(AParams: TStringList): string;
var
  Values: array of Double;
  i: Integer;
  Sum, Mean, Variance: Double;
  MinVal, MaxVal: Double;
begin
  if AParams.Count = 0 then
  begin
    Result := 'Error: ไม่มีข้อมูล';
    Exit;
  end;
  
  SetLength(Values, AParams.Count);
  for i := 0 to AParams.Count - 1 do
    Values[i] := StrToFloatDef(AParams[i], 0);
    
  // คำนวณค่าสถิติ
  Sum := 0;
  MinVal := Values[0];
  MaxVal := Values[0];
  
  for i := 0 to High(Values) do
  begin
    Sum := Sum + Values[i];
    if Values[i] < MinVal then MinVal := Values[i];
    if Values[i] > MaxVal then MaxVal := Values[i];
  end;
  
  Mean := Sum / Length(Values);
  
  // คำนวณ Variance
  Variance := 0;
  for i := 0 to High(Values) do
    Variance := Variance + Sqr(Values[i] - Mean);
  Variance := Variance / Length(Values);
  
  Result := Format('Count=%d, Sum=%.2f, Mean=%.2f, StdDev=%.2f, Min=%.2f, Max=%.2f',
    [Length(Values), Sum, Mean, Sqrt(Variance), MinVal, MaxVal]);
end;

// ฟังก์ชันที่ส่งออกจาก DLL
function GetPluginInfo: TPluginInfo; stdcall;
var
  Plugin: TCalcPlugin;
begin
  Plugin := TCalcPlugin.Create;
  try
    Result := Plugin.GetInfo;
  finally
    Plugin.Free;
  end;
end;

function CreatePlugin: IPlugin; stdcall;
begin
  Result := TCalcPlugin.Create;
end;

procedure DestroyPlugin(APlugin: IPlugin); stdcall;
begin
  APlugin := nil; // Interface จะ free ตัวเอง
end;

exports
  GetPluginInfo,
  CreatePlugin,
  DestroyPlugin;

begin
end.
```

---

## 61.3 Plugin Manager

### คลาสสำหรับจัดการ Plugins

```pascal
unit plugin_manager;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Dynlibs,
  plugin_interface;

type
  // ข้อมูลของ Plugin ที่โหลดแล้ว
  TLoadedPlugin = class
  private
    FHandle: TLibHandle;
    FPlugin: IPlugin;
    FInfo: TPluginInfo;
    FFilePath: string;
    FCreateFn: TCreatePlugin;
    FDestroyFn: TDestroyPlugin;
  public
    constructor Create(const AFilePath: string);
    destructor Destroy; override;
    
    property Handle: TLibHandle read FHandle;
    property Plugin: IPlugin read FPlugin;
    property Info: TPluginInfo read FInfo;
    property FilePath: string read FFilePath;
  end;

  // Event สำหรับแจ้งเหตุการณ์ Plugin
  TPluginEvent = procedure(const APlugin: TLoadedPlugin) of object;
  TPluginErrorEvent = procedure(const AFilePath, AError: string) of object;

  // Plugin Manager หลัก
  TPluginManager = class
  private
    FPlugins: TList;
    FSearchPaths: TStringList;
    FHost: TObject;
    FOnPluginLoaded: TPluginEvent;
    FOnPluginUnloaded: TPluginEvent;
    FOnError: TPluginErrorEvent;
    
    function GetPlugin(Index: Integer): TLoadedPlugin;
    function GetCount: Integer;
    procedure DoError(const AFilePath, AError: string);
  public
    constructor Create(AHost: TObject);
    destructor Destroy; override;
    
    // โหลด Plugin
    function LoadPlugin(const AFilePath: string): TLoadedPlugin;
    function LoadPluginsFromDirectory(const ADir: string; 
      const APattern: string = ''): Integer;
    
    // ปลด Plugin
    procedure UnloadPlugin(APlugin: TLoadedPlugin);
    procedure UnloadAll;
    
    // ค้นหา Plugin
    function FindPlugin(const AName: string): TLoadedPlugin;
    function FindPluginByFile(const AFilePath: string): TLoadedPlugin;
    
    // เรียกใช้ Plugin
    function ExecutePlugin(const AName, ACommand: string; 
      AParams: TStringList = nil): string;
    function ExecuteAll(const ACommand: string; 
      AParams: TStringList = nil): TStringList;
    
    // Hot-reload
    function ReloadPlugin(const AName: string): Boolean;
    procedure WatchForChanges(AInterval: Integer = 1000);
    
    property Count: Integer read GetCount;
    property Plugins[Index: Integer]: TLoadedPlugin read GetPlugin;
    property SearchPaths: TStringList read FSearchPaths;
    property OnPluginLoaded: TPluginEvent read FOnPluginLoaded write FOnPluginLoaded;
    property OnPluginUnloaded: TPluginEvent read FOnPluginUnloaded write FOnPluginUnloaded;
    property OnError: TPluginErrorEvent read FOnError write FOnError;
  end;

implementation

{ TLoadedPlugin }

constructor TLoadedPlugin.Create(const AFilePath: string);
var
  GetInfoFn: TGetPluginInfo;
begin
  inherited Create;
  FFilePath := AFilePath;
  FHandle := SafeLoadLibrary(AFilePath);
  
  if FHandle = NilHandle then
    raise Exception.CreateFmt('ไม่สามารถโหลด library: %s (%s)', 
      [AFilePath, GetLoadErrorStr]);
      
  // ดึงฟังก์ชันจาก DLL
  GetInfoFn := TGetPluginInfo(GetProcedureAddress(FHandle, 'GetPluginInfo'));
  FCreateFn := TCreatePlugin(GetProcedureAddress(FHandle, 'CreatePlugin'));
  FDestroyFn := TDestroyPlugin(GetProcedureAddress(FHandle, 'DestroyPlugin'));
  
  if not Assigned(FCreateFn) then
    raise Exception.CreateFmt('Library ไม่มีฟังก์ชัน CreatePlugin: %s', [AFilePath]);
    
  // ดึงข้อมูล Plugin
  if Assigned(GetInfoFn) then
    FInfo := GetInfoFn();
    
  // สร้าง Plugin instance
  FPlugin := FCreateFn();
  
  if not Assigned(FPlugin) then
    raise Exception.CreateFmt('ไม่สามารถสร้าง Plugin: %s', [AFilePath]);
end;

destructor TLoadedPlugin.Destroy;
begin
  if Assigned(FPlugin) then
  begin
    FPlugin.Finalize;
    if Assigned(FDestroyFn) then
      FDestroyFn(FPlugin);
    FPlugin := nil;
  end;
  
  if FHandle <> NilHandle then
  begin
    FreeLibrary(FHandle);
    FHandle := NilHandle;
  end;
  
  inherited Destroy;
end;

{ TPluginManager }

constructor TPluginManager.Create(AHost: TObject);
begin
  inherited Create;
  FHost := AHost;
  FPlugins := TList.Create;
  FSearchPaths := TStringList.Create;
end;

destructor TPluginManager.Destroy;
begin
  UnloadAll;
  FSearchPaths.Free;
  FPlugins.Free;
  inherited Destroy;
end;

function TPluginManager.GetPlugin(Index: Integer): TLoadedPlugin;
begin
  Result := TLoadedPlugin(FPlugins[Index]);
end;

function TPluginManager.GetCount: Integer;
begin
  Result := FPlugins.Count;
end;

procedure TPluginManager.DoError(const AFilePath, AError: string);
begin
  if Assigned(FOnError) then
    FOnError(AFilePath, AError)
  else
    WriteLn('Plugin Error [', AFilePath, ']: ', AError);
end;

function TPluginManager.LoadPlugin(const AFilePath: string): TLoadedPlugin;
var
  Plugin: TLoadedPlugin;
begin
  Result := nil;
  
  // ตรวจสอบว่าโหลดแล้วหรือยัง
  if Assigned(FindPluginByFile(AFilePath)) then
  begin
    DoError(AFilePath, 'Plugin นี้โหลดแล้ว');
    Exit;
  end;
  
  try
    Plugin := TLoadedPlugin.Create(AFilePath);
    
    // ตรวจสอบ API Version
    if Plugin.Info.APIVersion <> PLUGIN_API_VERSION then
    begin
      Plugin.Free;
      DoError(AFilePath, Format('API Version ไม่ตรงกัน (ต้องการ %d, ได้ %d)',
        [PLUGIN_API_VERSION, Plugin.Info.APIVersion]));
      Exit;
    end;
    
    // Initialize Plugin
    if not Plugin.Plugin.Initialize(FHost) then
    begin
      Plugin.Free;
      DoError(AFilePath, 'Initialize ล้มเหลว');
      Exit;
    end;
    
    FPlugins.Add(Plugin);
    
    if Assigned(FOnPluginLoaded) then
      FOnPluginLoaded(Plugin);
      
    Result := Plugin;
    WriteLn('โหลด Plugin สำเร็จ: ', Plugin.Info.Name, ' v', Plugin.Info.Version);
    
  except
    on E: Exception do
      DoError(AFilePath, E.Message);
  end;
end;

function TPluginManager.LoadPluginsFromDirectory(const ADir: string;
  const APattern: string): Integer;
var
  SR: TSearchRec;
  Pattern: string;
  FilePath: string;
begin
  Result := 0;
  
  if APattern <> '' then
    Pattern := APattern
  else
  begin
    {$IFDEF WINDOWS}
    Pattern := '*.dll';
    {$ELSE}
    Pattern := '*.so';
    {$ENDIF}
  end;
  
  if FindFirst(IncludeTrailingPathDelimiter(ADir) + Pattern, faAnyFile, SR) = 0 then
  begin
    try
      repeat
        FilePath := IncludeTrailingPathDelimiter(ADir) + SR.Name;
        if Assigned(LoadPlugin(FilePath)) then
          Inc(Result);
      until FindNext(SR) <> 0;
    finally
      FindClose(SR);
    end;
  end;
end;

procedure TPluginManager.UnloadPlugin(APlugin: TLoadedPlugin);
var
  Idx: Integer;
begin
  Idx := FPlugins.IndexOf(APlugin);
  if Idx >= 0 then
  begin
    if Assigned(FOnPluginUnloaded) then
      FOnPluginUnloaded(APlugin);
    FPlugins.Delete(Idx);
    APlugin.Free;
  end;
end;

procedure TPluginManager.UnloadAll;
var
  i: Integer;
begin
  for i := FPlugins.Count - 1 downto 0 do
    TLoadedPlugin(FPlugins[i]).Free;
  FPlugins.Clear;
end;

function TPluginManager.FindPlugin(const AName: string): TLoadedPlugin;
var
  i: Integer;
begin
  Result := nil;
  for i := 0 to FPlugins.Count - 1 do
    if SameText(TLoadedPlugin(FPlugins[i]).Info.Name, AName) then
    begin
      Result := TLoadedPlugin(FPlugins[i]);
      Exit;
    end;
end;

function TPluginManager.FindPluginByFile(const AFilePath: string): TLoadedPlugin;
var
  i: Integer;
begin
  Result := nil;
  for i := 0 to FPlugins.Count - 1 do
    if SameText(TLoadedPlugin(FPlugins[i]).FilePath, AFilePath) then
    begin
      Result := TLoadedPlugin(FPlugins[i]);
      Exit;
    end;
end;

function TPluginManager.ExecutePlugin(const AName, ACommand: string;
  AParams: TStringList): string;
var
  Plugin: TLoadedPlugin;
  Params: TStringList;
begin
  Plugin := FindPlugin(AName);
  if not Assigned(Plugin) then
  begin
    Result := 'Error: ไม่พบ Plugin ' + AName;
    Exit;
  end;
  
  if AParams = nil then
  begin
    Params := TStringList.Create;
    try
      Result := Plugin.Plugin.Execute(ACommand, Params);
    finally
      Params.Free;
    end;
  end
  else
    Result := Plugin.Plugin.Execute(ACommand, AParams);
end;

function TPluginManager.ExecuteAll(const ACommand: string;
  AParams: TStringList): TStringList;
var
  i: Integer;
  Plugin: TLoadedPlugin;
  Params: TStringList;
  OwnParams: Boolean;
begin
  Result := TStringList.Create;
  OwnParams := AParams = nil;
  
  if OwnParams then
    Params := TStringList.Create
  else
    Params := AParams;
    
  try
    for i := 0 to FPlugins.Count - 1 do
    begin
      Plugin := TLoadedPlugin(FPlugins[i]);
      try
        Result.Add(Plugin.Info.Name + '=' + Plugin.Plugin.Execute(ACommand, Params));
      except
        on E: Exception do
          Result.Add(Plugin.Info.Name + '=Error: ' + E.Message);
      end;
    end;
  finally
    if OwnParams then
      Params.Free;
  end;
end;

function TPluginManager.ReloadPlugin(const AName: string): Boolean;
var
  Plugin: TLoadedPlugin;
  FilePath: string;
begin
  Result := False;
  Plugin := FindPlugin(AName);
  
  if not Assigned(Plugin) then
    Exit;
    
  FilePath := Plugin.FilePath;
  UnloadPlugin(Plugin);
  Result := Assigned(LoadPlugin(FilePath));
end;

procedure TPluginManager.WatchForChanges(AInterval: Integer);
begin
  // ใช้ TTimer หรือ Thread สำหรับตรวจสอบการเปลี่ยนแปลงไฟล์
  // Implementation ขึ้นอยู่กับ context (VCL/LCL หรือ Console)
  WriteLn('WatchForChanges: ตั้งค่า interval =', AInterval, 'ms');
end;

end.
```

---

## 61.4 โปรแกรมหลักที่ใช้ Plugin

```pascal
program plugin_host;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils,
  plugin_interface,
  plugin_manager;

type
  // Host Application
  TApplication = class
  private
    FPluginManager: TPluginManager;
    FConfig: TStringList;
    
    procedure OnPluginLoaded(const APlugin: TLoadedPlugin);
    procedure OnPluginUnloaded(const APlugin: TLoadedPlugin);
    procedure OnPluginError(const AFilePath, AError: string);
  public
    constructor Create;
    destructor Destroy; override;
    
    procedure Run;
    procedure ListPlugins;
    procedure RunCommand(const AInput: string);
  end;

{ TApplication }

constructor TApplication.Create;
begin
  inherited Create;
  FConfig := TStringList.Create;
  FPluginManager := TPluginManager.Create(Self);
  FPluginManager.OnPluginLoaded := @OnPluginLoaded;
  FPluginManager.OnPluginUnloaded := @OnPluginUnloaded;
  FPluginManager.OnError := @OnPluginError;
end;

destructor TApplication.Destroy;
begin
  FPluginManager.Free;
  FConfig.Free;
  inherited Destroy;
end;

procedure TApplication.OnPluginLoaded(const APlugin: TLoadedPlugin);
begin
  WriteLn('>>> Plugin โหลดแล้ว: ', APlugin.Info.Name);
end;

procedure TApplication.OnPluginUnloaded(const APlugin: TLoadedPlugin);
begin
  WriteLn('>>> Plugin ปลดแล้ว: ', APlugin.Info.Name);
end;

procedure TApplication.OnPluginError(const AFilePath, AError: string);
begin
  WriteLn('>>> Plugin Error [', ExtractFileName(AFilePath), ']: ', AError);
end;

procedure TApplication.ListPlugins;
var
  i: Integer;
  P: TLoadedPlugin;
  Info: TPluginInfo;
begin
  WriteLn('');
  WriteLn('=== Plugins ที่โหลดอยู่ (', FPluginManager.Count, ' plugins) ===');
  
  for i := 0 to FPluginManager.Count - 1 do
  begin
    P := FPluginManager.Plugins[i];
    Info := P.Info;
    WriteLn(Format('  [%d] %s v%s by %s', [i+1, Info.Name, Info.Version, Info.Author]));
    WriteLn('      ', Info.Description);
    WriteLn('      File: ', ExtractFileName(P.FilePath));
  end;
  WriteLn('');
end;

procedure TApplication.RunCommand(const AInput: string);
var
  Parts: TStringList;
  Cmd, PluginName, Command: string;
  Params: TStringList;
  Result: string;
  i: Integer;
begin
  Parts := TStringList.Create;
  Params := TStringList.Create;
  try
    Parts.Delimiter := ' ';
    Parts.DelimitedText := Trim(AInput);
    
    if Parts.Count = 0 then Exit;
    
    Cmd := LowerCase(Parts[0]);
    
    if Cmd = 'list' then
      ListPlugins
      
    else if Cmd = 'load' then
    begin
      if Parts.Count < 2 then
        WriteLn('Usage: load <filepath>')
      else
        FPluginManager.LoadPlugin(Parts[1]);
    end
    
    else if Cmd = 'unload' then
    begin
      if Parts.Count < 2 then
        WriteLn('Usage: unload <pluginname>')
      else
      begin
        var P := FPluginManager.FindPlugin(Parts[1]);
        if Assigned(P) then
          FPluginManager.UnloadPlugin(P)
        else
          WriteLn('ไม่พบ Plugin: ', Parts[1]);
      end;
    end
    
    else if Cmd = 'reload' then
    begin
      if Parts.Count < 2 then
        WriteLn('Usage: reload <pluginname>')
      else
      begin
        if FPluginManager.ReloadPlugin(Parts[1]) then
          WriteLn('Reload สำเร็จ')
        else
          WriteLn('Reload ล้มเหลว');
      end;
    end
    
    else if Cmd = 'exec' then
    begin
      // exec <pluginname> <command> [params...]
      if Parts.Count < 3 then
        WriteLn('Usage: exec <pluginname> <command> [params...]')
      else
      begin
        PluginName := Parts[1];
        Command := Parts[2];
        
        for i := 3 to Parts.Count - 1 do
          Params.Add(Parts[i]);
          
        Result := FPluginManager.ExecutePlugin(PluginName, Command, Params);
        WriteLn('Result: ', Result);
      end;
    end
    
    else if Cmd = 'execall' then
    begin
      if Parts.Count < 2 then
        WriteLn('Usage: execall <command> [params...]')
      else
      begin
        Command := Parts[1];
        for i := 2 to Parts.Count - 1 do
          Params.Add(Parts[i]);
          
        var Results := FPluginManager.ExecuteAll(Command, Params);
        try
          for i := 0 to Results.Count - 1 do
            WriteLn('  ', Results[i]);
        finally
          Results.Free;
        end;
      end;
    end
    
    else if Cmd = 'loaddir' then
    begin
      if Parts.Count < 2 then
        WriteLn('Usage: loaddir <directory>')
      else
      begin
        var Count := FPluginManager.LoadPluginsFromDirectory(Parts[1]);
        WriteLn('โหลด Plugins จาก directory: ', Count, ' plugins');
      end;
    end
    
    else
      WriteLn('ไม่รู้จักคำสั่ง: ', Cmd);
      
  finally
    Parts.Free;
    Params.Free;
  end;
end;

procedure TApplication.Run;
var
  Input: string;
begin
  WriteLn('=== Plugin Host Application ===');
  WriteLn('คำสั่ง: list, load <file>, unload <name>, reload <name>');
  WriteLn('        exec <plugin> <cmd> [params], execall <cmd>, loaddir <dir>');
  WriteLn('        quit');
  WriteLn('');
  
  // โหลด Plugins จาก directory เริ่มต้น
  if DirectoryExists('plugins') then
  begin
    WriteLn('กำลังโหลด plugins จาก ./plugins/...');
    FPluginManager.LoadPluginsFromDirectory('plugins');
  end;
  
  // Main loop
  repeat
    Write('plugin> ');
    ReadLn(Input);
    Input := Trim(Input);
    
    if Input = '' then Continue;
    if LowerCase(Input) = 'quit' then Break;
    
    RunCommand(Input);
    
  until False;
  
  WriteLn('ออกจากโปรแกรม...');
end;

var
  App: TApplication;
begin
  App := TApplication.Create;
  try
    App.Run;
  finally
    App.Free;
  end;
end.
```

---

## 61.5 Hot-Reload System

```pascal
unit hot_reload;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils,
  plugin_interface,
  plugin_manager;

type
  TFileWatcher = class
  private
    FManager: TPluginManager;
    FFileTimes: TStringList;  // filename=modtime
    FCheckInterval: Integer;
    FActive: Boolean;
    FThread: TThread;
    
    procedure CheckFiles;
    function GetFileModTime(const AFilePath: string): TDateTime;
  public
    constructor Create(AManager: TPluginManager; AInterval: Integer = 2000);
    destructor Destroy; override;
    
    procedure Start;
    procedure Stop;
    
    property Active: Boolean read FActive;
  end;

  // Thread สำหรับตรวจสอบการเปลี่ยนแปลง
  TWatchThread = class(TThread)
  private
    FWatcher: TFileWatcher;
  protected
    procedure Execute; override;
  public
    constructor Create(AWatcher: TFileWatcher);
  end;

implementation

{ TWatchThread }

constructor TWatchThread.Create(AWatcher: TFileWatcher);
begin
  inherited Create(True);
  FWatcher := AWatcher;
  FreeOnTerminate := False;
end;

procedure TWatchThread.Execute;
begin
  while not Terminated do
  begin
    FWatcher.CheckFiles;
    Sleep(FWatcher.FCheckInterval);
  end;
end;

{ TFileWatcher }

constructor TFileWatcher.Create(AManager: TPluginManager; AInterval: Integer);
begin
  inherited Create;
  FManager := AManager;
  FFileTimes := TStringList.Create;
  FCheckInterval := AInterval;
  FActive := False;
end;

destructor TFileWatcher.Destroy;
begin
  Stop;
  FFileTimes.Free;
  inherited Destroy;
end;

function TFileWatcher.GetFileModTime(const AFilePath: string): TDateTime;
var
  Info: TSearchRec;
begin
  Result := 0;
  if FindFirst(AFilePath, faAnyFile, Info) = 0 then
  begin
    {$IFDEF WINDOWS}
    Result := FileDateToDateTime(Info.Time);
    {$ELSE}
    Result := FileDateToDateTime(Info.Time);
    {$ENDIF}
    FindClose(Info);
  end;
end;

procedure TFileWatcher.CheckFiles;
var
  i: Integer;
  Plugin: TLoadedPlugin;
  FilePath: string;
  CurrentTime, StoredTime: TDateTime;
  TimeStr: string;
begin
  // บันทึกเวลาแก้ไขปัจจุบัน
  for i := 0 to FManager.Count - 1 do
  begin
    Plugin := FManager.Plugins[i];
    FilePath := Plugin.FilePath;
    CurrentTime := GetFileModTime(FilePath);
    
    TimeStr := FormatDateTime('yyyymmddhhnnss', CurrentTime);
    
    if FFileTimes.IndexOfName(FilePath) < 0 then
    begin
      // เพิ่มครั้งแรก
      FFileTimes.Values[FilePath] := TimeStr;
    end
    else
    begin
      StoredTime := StrToDateTimeDef(FFileTimes.Values[FilePath], 0);
      
      if CurrentTime > StoredTime then
      begin
        // ไฟล์เปลี่ยนแปลง - reload
        WriteLn('ตรวจพบการเปลี่ยนแปลง: ', ExtractFileName(FilePath));
        WriteLn('กำลัง reload plugin: ', Plugin.Info.Name);
        
        FFileTimes.Values[FilePath] := TimeStr;
        
        // Reload ต้องทำใน main thread เพื่อความปลอดภัย
        // ที่นี่ใช้ Synchronize ถ้าเป็น VCL/LCL
        FManager.ReloadPlugin(Plugin.Info.Name);
      end;
    end;
  end;
end;

procedure TFileWatcher.Start;
begin
  if FActive then Exit;
  
  FActive := True;
  FThread := TWatchThread.Create(Self);
  TThread(FThread).Start;
  WriteLn('Hot-reload watcher เริ่มทำงาน');
end;

procedure TFileWatcher.Stop;
begin
  if not FActive then Exit;
  
  FActive := False;
  if Assigned(FThread) then
  begin
    TThread(FThread).Terminate;
    TThread(FThread).WaitFor;
    FThread.Free;
    FThread := nil;
  end;
  WriteLn('Hot-reload watcher หยุดทำงาน');
end;

end.
```

---

## 61.6 ตัวอย่างการใช้งานแบบ GUI (LCL)

```pascal
unit plugin_form;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics,
  StdCtrls, ComCtrls, ExtCtrls, Dialogs,
  plugin_interface, plugin_manager;

type
  TPluginManagerForm = class(TForm)
    pnlTop: TPanel;
    pnlBottom: TPanel;
    lvPlugins: TListView;
    memoLog: TMemo;
    btnLoad: TButton;
    btnUnload: TButton;
    btnReload: TButton;
    btnExec: TButton;
    edtCommand: TEdit;
    edtParams: TEdit;
    lblCommand: TLabel;
    lblParams: TLabel;
    splitter1: TSplitter;
    
    procedure FormCreate(Sender: TObject);
    procedure FormDestroy(Sender: TObject);
    procedure btnLoadClick(Sender: TObject);
    procedure btnUnloadClick(Sender: TObject);
    procedure btnReloadClick(Sender: TObject);
    procedure btnExecClick(Sender: TObject);
    procedure lvPluginsDblClick(Sender: TObject);
  private
    FManager: TPluginManager;
    
    procedure OnPluginLoaded(const APlugin: TLoadedPlugin);
    procedure OnPluginUnloaded(const APlugin: TLoadedPlugin);
    procedure OnPluginError(const AFilePath, AError: string);
    procedure RefreshPluginList;
    procedure Log(const AMsg: string);
    function GetSelectedPlugin: TLoadedPlugin;
  end;

implementation

{$R *.lfm}

procedure TPluginManagerForm.FormCreate(Sender: TObject);
begin
  FManager := TPluginManager.Create(Self);
  FManager.OnPluginLoaded := @OnPluginLoaded;
  FManager.OnPluginUnloaded := @OnPluginUnloaded;
  FManager.OnError := @OnPluginError;
  
  // ตั้งค่า ListView
  lvPlugins.ViewStyle := vsReport;
  with lvPlugins.Columns.Add do begin Name := 'ชื่อ Plugin'; Caption := 'ชื่อ Plugin'; Width := 150; end;
  with lvPlugins.Columns.Add do begin Name := 'Version'; Caption := 'Version'; Width := 70; end;
  with lvPlugins.Columns.Add do begin Name := 'Author'; Caption := 'Author'; Width := 100; end;
  with lvPlugins.Columns.Add do begin Name := 'Description'; Caption := 'Description'; Width := 200; end;
  
  Log('Plugin Manager เริ่มทำงาน');
end;

procedure TPluginManagerForm.FormDestroy(Sender: TObject);
begin
  FManager.Free;
end;

procedure TPluginManagerForm.OnPluginLoaded(const APlugin: TLoadedPlugin);
begin
  RefreshPluginList;
  Log('โหลด Plugin: ' + APlugin.Info.Name + ' v' + APlugin.Info.Version);
end;

procedure TPluginManagerForm.OnPluginUnloaded(const APlugin: TLoadedPlugin);
begin
  RefreshPluginList;
  Log('ปลด Plugin: ' + APlugin.Info.Name);
end;

procedure TPluginManagerForm.OnPluginError(const AFilePath, AError: string);
begin
  Log('Error [' + ExtractFileName(AFilePath) + ']: ' + AError);
  ShowMessage('Plugin Error: ' + AError);
end;

procedure TPluginManagerForm.RefreshPluginList;
var
  i: Integer;
  P: TLoadedPlugin;
  Item: TListItem;
  Info: TPluginInfo;
begin
  lvPlugins.Items.BeginUpdate;
  try
    lvPlugins.Items.Clear;
    
    for i := 0 to FManager.Count - 1 do
    begin
      P := FManager.Plugins[i];
      Info := P.Info;
      
      Item := lvPlugins.Items.Add;
      Item.Caption := Info.Name;
      Item.SubItems.Add(Info.Version);
      Item.SubItems.Add(Info.Author);
      Item.SubItems.Add(Info.Description);
      Item.Data := P;
    end;
  finally
    lvPlugins.Items.EndUpdate;
  end;
end;

procedure TPluginManagerForm.Log(const AMsg: string);
begin
  memoLog.Lines.Add(FormatDateTime('[hh:nn:ss] ', Now) + AMsg);
  memoLog.SelStart := Length(memoLog.Text);
  memoLog.SelLength := 0;
end;

function TPluginManagerForm.GetSelectedPlugin: TLoadedPlugin;
begin
  Result := nil;
  if lvPlugins.Selected <> nil then
    Result := TLoadedPlugin(lvPlugins.Selected.Data);
end;

procedure TPluginManagerForm.btnLoadClick(Sender: TObject);
var
  OD: TOpenDialog;
begin
  OD := TOpenDialog.Create(nil);
  try
    OD.Title := 'เลือก Plugin File';
    {$IFDEF WINDOWS}
    OD.Filter := 'Plugin files (*.dll)|*.dll|All files (*.*)|*.*';
    {$ELSE}
    OD.Filter := 'Plugin files (*.so)|*.so|All files (*.*)|*.*';
    {$ENDIF}
    
    if OD.Execute then
      FManager.LoadPlugin(OD.FileName);
  finally
    OD.Free;
  end;
end;

procedure TPluginManagerForm.btnUnloadClick(Sender: TObject);
var
  P: TLoadedPlugin;
begin
  P := GetSelectedPlugin;
  if Assigned(P) then
  begin
    if MessageDlg('ยืนยัน', 'ต้องการปลด Plugin "' + P.Info.Name + '" หรือไม่?',
      mtConfirmation, [mbYes, mbNo], 0) = mrYes then
      FManager.UnloadPlugin(P);
  end
  else
    ShowMessage('กรุณาเลือก Plugin ก่อน');
end;

procedure TPluginManagerForm.btnReloadClick(Sender: TObject);
var
  P: TLoadedPlugin;
begin
  P := GetSelectedPlugin;
  if Assigned(P) then
  begin
    if FManager.ReloadPlugin(P.Info.Name) then
      Log('Reload สำเร็จ: ' + P.Info.Name)
    else
      Log('Reload ล้มเหลว');
  end
  else
    ShowMessage('กรุณาเลือก Plugin ก่อน');
end;

procedure TPluginManagerForm.btnExecClick(Sender: TObject);
var
  P: TLoadedPlugin;
  Params: TStringList;
  Result: string;
begin
  P := GetSelectedPlugin;
  if not Assigned(P) then
  begin
    ShowMessage('กรุณาเลือก Plugin ก่อน');
    Exit;
  end;
  
  if edtCommand.Text = '' then
  begin
    ShowMessage('กรุณาใส่คำสั่ง');
    Exit;
  end;
  
  Params := TStringList.Create;
  try
    Params.DelimitedText := edtParams.Text;
    Result := P.Plugin.Execute(edtCommand.Text, Params);
    Log(P.Info.Name + '.' + edtCommand.Text + ' => ' + Result);
  finally
    Params.Free;
  end;
end;

procedure TPluginManagerForm.lvPluginsDblClick(Sender: TObject);
var
  P: TLoadedPlugin;
  Info: TPluginInfo;
begin
  P := GetSelectedPlugin;
  if Assigned(P) then
  begin
    Info := P.Info;
    ShowMessage(Format('Plugin: %s%sVersion: %s%sAuthor: %s%sDescription: %s%sFile: %s',
      [Info.Name, LineEnding, Info.Version, LineEnding, 
       Info.Author, LineEnding, Info.Description, LineEnding, P.FilePath]));
  end;
end;

end.
```

---

## 61.7 สรุปและแนวทางปฏิบัติที่ดี

### การออกแบบ Plugin Interface ที่ดี
1. ใช้ Interface แทน Class เพื่อความยืดหยุ่น
2. กำหนด API Version เพื่อจัดการความเข้ากันได้
3. มี Initialize/Finalize สำหรับจัดการทรัพยากร
4. ส่งออกฟังก์ชันในรูปแบบ C calling convention (stdcall)

### ข้อควรระวัง
```pascal
// ❌ อย่าแชร์ Object ข้าม DLL boundary โดยตรง
// ✅ ใช้ Interface หรือ Pointer

// ❌ อย่าส่ง String ของ Delphi/FPC ข้าม DLL
// ✅ ใช้ PChar หรือ PAnsiChar

// ตัวอย่างการส่ง string อย่างปลอดภัย
type
  TGetNameProc = function(Buffer: PAnsiChar; BufferSize: Integer): Integer; stdcall;

function TMyPlugin.GetName(Buffer: PAnsiChar; BufferSize: Integer): Integer; stdcall;
var
  S: string;
begin
  S := 'My Plugin Name';
  Result := Min(Length(S), BufferSize - 1);
  Move(S[1], Buffer^, Result);
  Buffer[Result] := #0;
end;
```

### การจัดการ Memory
```pascal
// ใช้ Reference Counting ผ่าน Interface
type
  IMyPlugin = interface
    // Interface ใช้ Reference Counting อัตโนมัติ
    // ไม่ต้อง Free เอง
  end;

// Plugin สืบทอดจาก TInterfacedObject
TMyPlugin = class(TInterfacedObject, IMyPlugin)
  // RefCount จัดการโดยอัตโนมัติ
end;
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:
- การสร้าง Plugin Interface มาตรฐาน
- การสร้าง DLL/Shared Library สำหรับ Plugin
- Plugin Manager สำหรับจัดการหลาย Plugins
- Hot-reload เพื่ออัปเดต Plugin โดยไม่หยุดโปรแกรม
- การใช้งาน Plugin ใน GUI Application

Plugin Architecture ช่วยให้โปรแกรมมีความยืดหยุ่นสูง สามารถขยายฟังก์ชันได้โดยไม่ต้องแก้ไขโปรแกรมหลัก เหมาะสำหรับระบบที่ต้องการความสามารถในการ customize สูง
