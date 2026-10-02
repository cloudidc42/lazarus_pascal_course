# Part 45 - Windows Registry ใน Lazarus/Pascal

## บทนำ

Windows Registry เป็นฐานข้อมูลที่เก็บ configuration settings ของ Windows และแอปพลิเคชันต่างๆ Lazarus/FPC มี `TRegistry` class ใน unit `Registry` สำหรับจัดการ Registry อย่างครบถ้วน

> **หมายเหตุ**: Registry เป็น Windows-specific feature ถ้าต้องการ cross-platform ให้ใช้ `TIniFile` หรือ JSON/XML config files แทน

---

## 45.1 Registry Overview

### โครงสร้าง Registry

```
Registry Root Keys:
│
├── HKEY_CLASSES_ROOT (HKCR)
│   └── File associations, COM registrations
│
├── HKEY_CURRENT_USER (HKCU)
│   ├── Software\
│   │   └── YourApp\
│   │       └── Settings (user-specific settings)
│   └── Environment
│
├── HKEY_LOCAL_MACHINE (HKLM)
│   ├── Software\
│   │   └── YourApp\ (machine-wide settings)
│   ├── System\
│   └── Hardware\
│
├── HKEY_USERS (HKU)
│   └── .DEFAULT
│
└── HKEY_CURRENT_CONFIG (HKCC)
```

### Value Types

| Type | Description | Pascal Type |
|------|-------------|-------------|
| REG_SZ | String | string |
| REG_DWORD | 32-bit integer | DWORD |
| REG_QWORD | 64-bit integer | int64 |
| REG_BINARY | Binary data | TBytes |
| REG_MULTI_SZ | Multiple strings | TStringList |
| REG_EXPAND_SZ | Expandable string | string |

---

## 45.2 TRegistry Class

### การใช้งาน TRegistry พื้นฐาน

```pascal
program RegistryBasics;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils;

procedure DemoBasicRegistry;
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create;
  try
    // กำหนด root key (default คือ HKEY_CURRENT_USER)
    Reg.RootKey := HKEY_CURRENT_USER;
    
    // เปิด/สร้าง key
    if Reg.OpenCreateKey('Software\TestApp\Settings') then
    begin
      try
        // เขียนค่า
        Reg.WriteString('AppName', 'My Test Application');
        Reg.WriteInteger('MaxConnections', 10);
        Reg.WriteBool('EnableLogging', True);
        Reg.WriteFloat('Version', 2.5);
        
        WriteLn('Values written to registry');
        
        // อ่านค่า
        WriteLn('AppName: ', Reg.ReadString('AppName'));
        WriteLn('MaxConnections: ', Reg.ReadInteger('MaxConnections'));
        WriteLn('EnableLogging: ', Reg.ReadBool('EnableLogging'));
        WriteLn('Version: ', Reg.ReadFloat('Version'):0:1);
        
      finally
        Reg.CloseKey;
      end;
    end
    else
      WriteLn('Cannot open/create registry key');
  finally
    Reg.Free;
  end;
end;

begin
  WriteLn('=== Registry Basics Demo ===');
  WriteLn('');
  DemoBasicRegistry;
end.

{$ELSE}
begin
  WriteLn('Registry is Windows-only');
end.
{$ENDIF}
```

---

## 45.3 Reading Registry Values

### การอ่านค่าจาก Registry

```pascal
program ReadRegistryValues;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils, Classes;

type
  TRegistryReader = class
  private
    FReg: TRegistry;
  public
    constructor Create(RootKey: HKEY; const KeyPath: string);
    destructor Destroy; override;
    
    function ReadString(const Name, Default: string): string;
    function ReadInt(const Name: string; Default: integer): integer;
    function ReadBool(const Name: string; Default: boolean): boolean;
    function ReadFloat(const Name: string; Default: double): double;
    function ReadStrings(const Name: string; Strings: TStringList): boolean;
    function ReadBinary(const Name: string; var Buffer: TBytes): boolean;
    function ValueExists(const Name: string): boolean;
    procedure GetValueNames(Names: TStringList);
  end;

constructor TRegistryReader.Create(RootKey: HKEY; const KeyPath: string);
begin
  inherited Create;
  FReg := TRegistry.Create(KEY_READ);
  FReg.RootKey := RootKey;
  
  if not FReg.OpenKey(KeyPath, False) then
  begin
    FReg.Free;
    FReg := nil;
    raise Exception.CreateFmt('Cannot open registry key: %s', [KeyPath]);
  end;
end;

destructor TRegistryReader.Destroy;
begin
  if Assigned(FReg) then
  begin
    FReg.CloseKey;
    FReg.Free;
  end;
  inherited Destroy;
end;

function TRegistryReader.ReadString(const Name, Default: string): string;
begin
  if Assigned(FReg) and FReg.ValueExists(Name) then
    Result := FReg.ReadString(Name)
  else
    Result := Default;
end;

function TRegistryReader.ReadInt(const Name: string; Default: integer): integer;
begin
  if Assigned(FReg) and FReg.ValueExists(Name) then
    Result := FReg.ReadInteger(Name)
  else
    Result := Default;
end;

function TRegistryReader.ReadBool(const Name: string; Default: boolean): boolean;
begin
  if Assigned(FReg) and FReg.ValueExists(Name) then
    Result := FReg.ReadBool(Name)
  else
    Result := Default;
end;

function TRegistryReader.ReadFloat(const Name: string; Default: double): double;
begin
  if Assigned(FReg) and FReg.ValueExists(Name) then
  begin
    try
      Result := FReg.ReadFloat(Name);
    except
      Result := Default;
    end;
  end
  else
    Result := Default;
end;

function TRegistryReader.ReadStrings(const Name: string; Strings: TStringList): boolean;
begin
  Result := False;
  if not Assigned(FReg) then Exit;
  if not FReg.ValueExists(Name) then Exit;
  
  try
    FReg.ReadStringList(Name, Strings);
    Result := True;
  except
    Result := False;
  end;
end;

function TRegistryReader.ReadBinary(const Name: string; var Buffer: TBytes): boolean;
var
  DataType: TRegDataType;
  DataSize: integer;
begin
  Result := False;
  if not Assigned(FReg) then Exit;
  if not FReg.ValueExists(Name) then Exit;
  
  try
    DataSize := FReg.GetDataSize(Name);
    if DataSize > 0 then
    begin
      SetLength(Buffer, DataSize);
      FReg.ReadBinaryData(Name, Buffer[0], DataSize);
      Result := True;
    end;
  except
    Result := False;
  end;
end;

function TRegistryReader.ValueExists(const Name: string): boolean;
begin
  Result := Assigned(FReg) and FReg.ValueExists(Name);
end;

procedure TRegistryReader.GetValueNames(Names: TStringList);
begin
  if Assigned(FReg) then
    FReg.GetValueNames(Names);
end;

// อ่านข้อมูลระบบจาก Registry
procedure ReadSystemInfo;
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create(KEY_READ);
  try
    Reg.RootKey := HKEY_LOCAL_MACHINE;
    
    // อ่าน Windows version
    if Reg.OpenKey('SOFTWARE\Microsoft\Windows NT\CurrentVersion', False) then
    begin
      try
        WriteLn('=== Windows Version Info ===');
        WriteLn('ProductName: ', Reg.ReadString('ProductName'));
        WriteLn('DisplayVersion: ', Reg.ReadString('DisplayVersion'));
        WriteLn('BuildLab: ', Reg.ReadString('BuildLab'));
        WriteLn('RegisteredOwner: ', Reg.ReadString('RegisteredOwner'));
        WriteLn('InstallDate: ', Reg.ReadInteger('InstallDate'));
      finally
        Reg.CloseKey;
      end;
    end;
    
    WriteLn('');
    
    // อ่าน CPU info
    if Reg.OpenKey('HARDWARE\DESCRIPTION\System\CentralProcessor\0', False) then
    begin
      try
        WriteLn('=== CPU Info ===');
        WriteLn('ProcessorName: ', Reg.ReadString('ProcessorNameString'));
        WriteLn('Identifier: ', Reg.ReadString('Identifier'));
        WriteLn('VendorId: ', Reg.ReadString('VendorIdentifier'));
        WriteLn('MHz: ', Reg.ReadInteger('~MHz'));
      finally
        Reg.CloseKey;
      end;
    end;
    
  finally
    Reg.Free;
  end;
end;

begin
  WriteLn('=== Reading Registry Values ===');
  WriteLn('');
  ReadSystemInfo;
end.
{$ELSE}
begin
  WriteLn('Registry reading is Windows-only');
end.
{$ENDIF}
```

---

## 45.4 Writing Registry Values

### การเขียนค่าลง Registry

```pascal
program WriteRegistryValues;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils, Classes;

type
  TAppSettings = class
  private
    FAppPath: string;
    FRootKey: HKEY;
    FKeyPath: string;
    
    function OpenKey(Write: boolean): TRegistry;
  public
    constructor Create(const AppName: string; UserScope: boolean = True);
    
    procedure WriteString(const Name, Value: string);
    procedure WriteInteger(const Name: string; Value: integer);
    procedure WriteBool(const Name: string; Value: boolean);
    procedure WriteFloat(const Name: string; Value: double);
    procedure WriteStrings(const Name: string; Values: TStringList);
    procedure WriteBinary(const Name: string; const Buffer: TBytes);
    
    function ReadString(const Name, Default: string): string;
    function ReadInteger(const Name: string; Default: integer): integer;
    function ReadBool(const Name: string; Default: boolean): boolean;
    function ReadFloat(const Name: string; Default: double): double;
    
    procedure DeleteValue(const Name: string);
    procedure DeleteSection(const Section: string);
    procedure SaveAll(const KeyValues: array of string);
  end;

constructor TAppSettings.Create(const AppName: string; UserScope: boolean);
begin
  inherited Create;
  if UserScope then
    FRootKey := HKEY_CURRENT_USER
  else
    FRootKey := HKEY_LOCAL_MACHINE;
    
  FKeyPath := 'Software\' + AppName;
end;

function TAppSettings.OpenKey(Write: boolean): TRegistry;
var
  Access: REGSAM;
begin
  if Write then
    Access := KEY_ALL_ACCESS
  else
    Access := KEY_READ;
    
  Result := TRegistry.Create(Access);
  Result.RootKey := FRootKey;
  
  if Write then
  begin
    if not Result.OpenCreateKey(FKeyPath) then
    begin
      Result.Free;
      raise Exception.Create('Cannot open/create registry key: ' + FKeyPath);
    end;
  end
  else
  begin
    if not Result.OpenKey(FKeyPath, False) then
    begin
      Result.Free;
      Result := nil;
    end;
  end;
end;

procedure TAppSettings.WriteString(const Name, Value: string);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    Reg.WriteString(Name, Value);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.WriteInteger(const Name: string; Value: integer);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    Reg.WriteInteger(Name, Value);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.WriteBool(const Name: string; Value: boolean);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    Reg.WriteBool(Name, Value);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.WriteFloat(const Name: string; Value: double);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    Reg.WriteFloat(Name, Value);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.WriteStrings(const Name: string; Values: TStringList);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    Reg.WriteStringList(Name, Values);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.WriteBinary(const Name: string; const Buffer: TBytes);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  try
    if Length(Buffer) > 0 then
      Reg.WriteBinaryData(Name, Buffer[0], Length(Buffer));
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

function TAppSettings.ReadString(const Name, Default: string): string;
var
  Reg: TRegistry;
begin
  Result := Default;
  Reg := OpenKey(False);
  if not Assigned(Reg) then Exit;
  
  try
    if Reg.ValueExists(Name) then
      Result := Reg.ReadString(Name);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

function TAppSettings.ReadInteger(const Name: string; Default: integer): integer;
var
  Reg: TRegistry;
begin
  Result := Default;
  Reg := OpenKey(False);
  if not Assigned(Reg) then Exit;
  
  try
    if Reg.ValueExists(Name) then
      Result := Reg.ReadInteger(Name);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

function TAppSettings.ReadBool(const Name: string; Default: boolean): boolean;
var
  Reg: TRegistry;
begin
  Result := Default;
  Reg := OpenKey(False);
  if not Assigned(Reg) then Exit;
  
  try
    if Reg.ValueExists(Name) then
      Result := Reg.ReadBool(Name);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

function TAppSettings.ReadFloat(const Name: string; Default: double): double;
var
  Reg: TRegistry;
begin
  Result := Default;
  Reg := OpenKey(False);
  if not Assigned(Reg) then Exit;
  
  try
    if Reg.ValueExists(Name) then
    begin
      try
        Result := Reg.ReadFloat(Name);
      except
        Result := Default;
      end;
    end;
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.DeleteValue(const Name: string);
var
  Reg: TRegistry;
begin
  Reg := OpenKey(True);
  if not Assigned(Reg) then Exit;
  
  try
    if Reg.ValueExists(Name) then
      Reg.DeleteValue(Name);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TAppSettings.DeleteSection(const Section: string);
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := FRootKey;
    Reg.DeleteKey(FKeyPath + '\' + Section);
  finally
    Reg.Free;
  end;
end;

procedure TAppSettings.SaveAll(const KeyValues: array of string);
var
  Reg: TRegistry;
  i: integer;
begin
  if Length(KeyValues) mod 2 <> 0 then
    raise Exception.Create('KeyValues must have even number of elements');
  
  Reg := OpenKey(True);
  try
    i := 0;
    while i < Length(KeyValues) do
    begin
      Reg.WriteString(KeyValues[i], KeyValues[i + 1]);
      Inc(i, 2);
    end;
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

var
  Settings: TAppSettings;
  RecentFiles: TStringList;
  BinaryData: TBytes;
begin
  WriteLn('=== Writing Registry Values Demo ===');
  WriteLn('');
  
  Settings := TAppSettings.Create('TestApp\Demo');
  try
    // เขียนค่าต่างๆ
    Settings.WriteString('AppVersion', '2.0.1');
    Settings.WriteString('LastUser', 'admin');
    Settings.WriteInteger('MaxRecentFiles', 10);
    Settings.WriteBool('AutoSave', True);
    Settings.WriteFloat('ZoomLevel', 1.5);
    
    // เขียน string list (recent files)
    RecentFiles := TStringList.Create;
    try
      RecentFiles.Add('C:\Documents\report1.docx');
      RecentFiles.Add('C:\Documents\report2.docx');
      RecentFiles.Add('D:\Projects\project.lpr');
      Settings.WriteStrings('RecentFiles', RecentFiles);
    finally
      RecentFiles.Free;
    end;
    
    // เขียน binary data (ตัวอย่าง: encrypted password hash)
    SetLength(BinaryData, 16);
    FillChar(BinaryData[0], 16, $AB);
    BinaryData[0] := $DE;
    BinaryData[1] := $AD;
    BinaryData[2] := $BE;
    BinaryData[3] := $EF;
    Settings.WriteBinary('AuthToken', BinaryData);
    
    WriteLn('All values written successfully!');
    WriteLn('');
    
    // อ่านกลับมา
    WriteLn('=== Reading back ===');
    WriteLn('AppVersion: ', Settings.ReadString('AppVersion', ''));
    WriteLn('LastUser: ', Settings.ReadString('LastUser', ''));
    WriteLn('MaxRecentFiles: ', Settings.ReadInteger('MaxRecentFiles', 0));
    WriteLn('AutoSave: ', Settings.ReadBool('AutoSave', False));
    WriteLn('ZoomLevel: ', Settings.ReadFloat('ZoomLevel', 1.0):0:1);
    
    // Cleanup - ลบ test registry keys
    WriteLn('');
    WriteLn('Cleaning up...');
    Settings.DeleteValue('AppVersion');
    Settings.DeleteValue('LastUser');
    Settings.DeleteValue('MaxRecentFiles');
    Settings.DeleteValue('AutoSave');
    Settings.DeleteValue('ZoomLevel');
    Settings.DeleteValue('RecentFiles');
    Settings.DeleteValue('AuthToken');
    WriteLn('Cleanup done');
    
  finally
    Settings.Free;
  end;
  
  // ลบ registry key ที่สร้างขึ้น
  var Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CURRENT_USER;
    Reg.DeleteKey('Software\TestApp\Demo');
    Reg.DeleteKey('Software\TestApp');
  finally
    Reg.Free;
  end;
end.
{$ELSE}
begin
  WriteLn('Registry writing is Windows-only');
end.
{$ENDIF}
```

---

## 45.5 Creating Keys / Deleting Keys / Enumerating

### การจัดการ Keys

```pascal
program ManageRegistryKeys;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils, Classes;

procedure EnumerateKeys(const KeyPath: string; RootKey: HKEY; Indent: integer = 0);
var
  Reg: TRegistry;
  SubKeys: TStringList;
  ValueNames: TStringList;
  IndentStr: string;
  Key: string;
  ValueName: string;
  DataType: TRegDataType;
  Value: string;
begin
  Reg := TRegistry.Create(KEY_READ);
  SubKeys := TStringList.Create;
  ValueNames := TStringList.Create;
  
  IndentStr := StringOfChar(' ', Indent * 2);
  
  try
    Reg.RootKey := RootKey;
    
    if Reg.OpenKey(KeyPath, False) then
    begin
      // แสดง values ใน key นี้
      Reg.GetValueNames(ValueNames);
      for ValueName in ValueNames do
      begin
        DataType := Reg.GetDataType(ValueName);
        case DataType of
          rdString, rdExpandString:
            Value := '"' + Reg.ReadString(ValueName) + '"';
          rdInteger:
            Value := IntToStr(Reg.ReadInteger(ValueName));
          rdBinary:
            Value := '(binary data)';
          else
            Value := '(unknown type)';
        end;
        
        WriteLn(IndentStr, '  [', ValueName, '] = ', Value);
      end;
      
      // แสดง sub-keys
      Reg.GetKeyNames(SubKeys);
      Reg.CloseKey;
      
      for Key in SubKeys do
      begin
        WriteLn(IndentStr, '+ ', Key);
        EnumerateKeys(KeyPath + '\' + Key, RootKey, Indent + 1);
      end;
    end;
  finally
    Reg.Free;
    SubKeys.Free;
    ValueNames.Free;
  end;
end;

procedure CreateComplexKeyStructure;
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CURRENT_USER;
    
    // สร้าง key structure
    if Reg.OpenCreateKey('Software\MyCompany\MyApp') then
    begin
      Reg.WriteString('AppName', 'My Application');
      Reg.WriteString('Version', '3.0.0');
      Reg.WriteInteger('BuildNumber', 1234);
      Reg.CloseKey;
    end;
    
    if Reg.OpenCreateKey('Software\MyCompany\MyApp\Database') then
    begin
      Reg.WriteString('Server', 'db.myapp.com');
      Reg.WriteInteger('Port', 5432);
      Reg.WriteString('DatabaseName', 'myapp_prod');
      Reg.WriteBool('UseSSL', True);
      Reg.CloseKey;
    end;
    
    if Reg.OpenCreateKey('Software\MyCompany\MyApp\UI') then
    begin
      Reg.WriteString('Theme', 'Dark');
      Reg.WriteString('Language', 'th-TH');
      Reg.WriteInteger('FontSize', 14);
      Reg.WriteInteger('WindowX', 100);
      Reg.WriteInteger('WindowY', 100);
      Reg.WriteInteger('WindowWidth', 1200);
      Reg.WriteInteger('WindowHeight', 800);
      Reg.CloseKey;
    end;
    
    if Reg.OpenCreateKey('Software\MyCompany\MyApp\Plugins') then
    begin
      Reg.WriteString('Plugin1', 'analytics.dll');
      Reg.WriteString('Plugin2', 'export.dll');
      Reg.WriteBool('AutoLoad', True);
      Reg.CloseKey;
    end;
    
    WriteLn('Key structure created');
    
  finally
    Reg.Free;
  end;
end;

procedure DeleteKeyRecursive(const KeyPath: string; RootKey: HKEY);
var
  Reg: TRegistry;
  SubKeys: TStringList;
  Key: string;
begin
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  SubKeys := TStringList.Create;
  
  try
    Reg.RootKey := RootKey;
    
    if Reg.OpenKey(KeyPath, False) then
    begin
      Reg.GetKeyNames(SubKeys);
      Reg.CloseKey;
      
      // ลบ sub-keys ก่อน (recursive)
      for Key in SubKeys do
        DeleteKeyRecursive(KeyPath + '\' + Key, RootKey);
    end;
    
    // ลบ key นี้
    if not Reg.DeleteKey(KeyPath) then
      WriteLn('Warning: Could not delete key: ', KeyPath);
      
  finally
    Reg.Free;
    SubKeys.Free;
  end;
end;

begin
  WriteLn('=== Registry Key Management Demo ===');
  WriteLn('');
  
  // สร้าง structure
  WriteLn('Creating key structure...');
  CreateComplexKeyStructure;
  WriteLn('');
  
  // แสดง structure
  WriteLn('Enumerating keys:');
  WriteLn('HKCU\Software\MyCompany\MyApp');
  EnumerateKeys('Software\MyCompany\MyApp', HKEY_CURRENT_USER);
  WriteLn('');
  
  // ลบ keys
  WriteLn('Deleting keys...');
  DeleteKeyRecursive('Software\MyCompany\MyApp', HKEY_CURRENT_USER);
  DeleteKeyRecursive('Software\MyCompany', HKEY_CURRENT_USER);
  WriteLn('Keys deleted');
end.
{$ELSE}
begin
  WriteLn('Registry key management is Windows-only');
end.
{$ENDIF}
```

---

## 45.6 Registry Monitoring

### การ Monitor การเปลี่ยนแปลง Registry

```pascal
program RegistryMonitor;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Windows, Registry, SysUtils, Classes;

type
  TRegistryChangeEvent = procedure(const KeyPath: string; ChangeType: integer) of object;
  
  TRegistryWatcher = class(TThread)
  private
    FKeyPath: string;
    FRootKey: HKEY;
    FNotifyFilter: DWORD;
    FOnChange: TRegistryChangeEvent;
    FKeyHandle: HKEY;
    FEventHandle: HANDLE;
    
  protected
    procedure Execute; override;
  public
    constructor Create(const KeyPath: string; RootKey: HKEY; 
      OnChange: TRegistryChangeEvent);
    destructor Destroy; override;
    
    procedure Stop;
  end;

constructor TRegistryWatcher.Create(const KeyPath: string; RootKey: HKEY;
  OnChange: TRegistryChangeEvent);
begin
  inherited Create(True);
  FKeyPath := KeyPath;
  FRootKey := RootKey;
  FOnChange := OnChange;
  FKeyHandle := 0;
  FEventHandle := 0;
  FreeOnTerminate := False;
  
  FNotifyFilter := 
    REG_NOTIFY_CHANGE_NAME or       // key name changes
    REG_NOTIFY_CHANGE_ATTRIBUTES or // attribute changes
    REG_NOTIFY_CHANGE_LAST_SET or   // value changes
    REG_NOTIFY_CHANGE_SECURITY;     // security descriptor changes
end;

destructor TRegistryWatcher.Destroy;
begin
  Stop;
  inherited Destroy;
end;

procedure TRegistryWatcher.Stop;
begin
  Terminate;
  if FEventHandle <> 0 then
    SetEvent(FEventHandle);
end;

procedure TRegistryWatcher.Execute;
var
  Result: DWORD;
begin
  // เปิด registry key
  if RegOpenKeyEx(FRootKey, PChar(FKeyPath), 0, KEY_NOTIFY, FKeyHandle) <> ERROR_SUCCESS then
  begin
    WriteLn('[Watcher] Cannot open key: ', FKeyPath);
    Exit;
  end;
  
  // สร้าง event สำหรับ notification
  FEventHandle := CreateEvent(nil, False, False, nil);
  
  try
    WriteLn('[Watcher] Monitoring: ', FKeyPath);
    
    while not Terminated do
    begin
      // ลงทะเบียน notification
      if RegNotifyChangeKeyValue(
        FKeyHandle,
        True,           // ติดตาม sub-keys ด้วย
        FNotifyFilter,
        FEventHandle,
        True           // asynchronous
      ) <> ERROR_SUCCESS then
        Break;
      
      // รอ event หรือ termination
      Result := WaitForSingleObject(FEventHandle, 1000);
      
      if Result = WAIT_OBJECT_0 then
      begin
        // เกิดการเปลี่ยนแปลง
        if Assigned(FOnChange) then
          Synchronize(procedure
          begin
            FOnChange(FKeyPath, 0);
          end);
      end;
    end;
  finally
    RegCloseKey(FKeyHandle);
    CloseHandle(FEventHandle);
    FKeyHandle := 0;
    FEventHandle := 0;
  end;
end;

procedure OnRegistryChanged(const KeyPath: string; ChangeType: integer);
begin
  WriteLn('[Change Detected] Key: ', KeyPath, ' at ', TimeToStr(Now));
end;

var
  Watcher: TRegistryWatcher;
  Reg: TRegistry;
  TestKey: string;
  i: integer;
begin
  WriteLn('=== Registry Monitor Demo ===');
  WriteLn('');
  
  TestKey := 'Software\TestApp\MonitorTest';
  
  // สร้าง test key
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CURRENT_USER;
    Reg.OpenCreateKey(TestKey);
    Reg.WriteString('InitialValue', 'Hello');
    Reg.CloseKey;
  finally
    Reg.Free;
  end;
  
  // เริ่ม monitoring
  Watcher := TRegistryWatcher.Create(
    TestKey, 
    HKEY_CURRENT_USER, 
    @OnRegistryChanged
  );
  try
    Watcher.Start;
    Sleep(200);
    
    // ทำการเปลี่ยนแปลงเพื่อทดสอบ
    Reg := TRegistry.Create(KEY_ALL_ACCESS);
    try
      Reg.RootKey := HKEY_CURRENT_USER;
      
      for i := 1 to 3 do
      begin
        Sleep(500);
        Reg.OpenKey(TestKey, False);
        Reg.WriteString('Value', 'Change ' + IntToStr(i));
        Reg.CloseKey;
        WriteLn('[Test] Changed value #', i);
      end;
    finally
      Reg.Free;
    end;
    
    Sleep(500);
    Watcher.Stop;
    Watcher.WaitFor;
    
  finally
    Watcher.Free;
  end;
  
  // ลบ test key
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CURRENT_USER;
    Reg.DeleteKey(TestKey);
    Reg.DeleteKey('Software\TestApp');
  finally
    Reg.Free;
  end;
  
  WriteLn('Done');
end.
{$ELSE}
begin
  WriteLn('Registry monitoring is Windows-only');
end.
{$ENDIF}
```

---

## 45.7 Auto-Start Applications

### การตั้งค่า Auto-Start

```pascal
program AutoStartApp;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils;

const
  RUN_KEY = 'Software\Microsoft\Windows\CurrentVersion\Run';

type
  TAutoStart = class
  public
    class procedure Enable(const AppName, ExePath: string; UserLevel: boolean = True);
    class procedure Disable(const AppName: string; UserLevel: boolean = True);
    class function IsEnabled(const AppName: string; UserLevel: boolean = True): boolean;
    class function GetStartupPath(const AppName: string; UserLevel: boolean = True): string;
    class procedure ListAllStartupItems;
  end;

class procedure TAutoStart.Enable(const AppName, ExePath: string; UserLevel: boolean);
var
  Reg: TRegistry;
  RootKey: HKEY;
begin
  if UserLevel then
    RootKey := HKEY_CURRENT_USER
  else
    RootKey := HKEY_LOCAL_MACHINE;
    
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := RootKey;
    if Reg.OpenKey(RUN_KEY, True) then
    begin
      try
        Reg.WriteString(AppName, '"' + ExePath + '"');
        WriteLn('Auto-start enabled for: ', AppName);
        WriteLn('Path: ', ExePath);
      finally
        Reg.CloseKey;
      end;
    end;
  finally
    Reg.Free;
  end;
end;

class procedure TAutoStart.Disable(const AppName: string; UserLevel: boolean);
var
  Reg: TRegistry;
  RootKey: HKEY;
begin
  if UserLevel then
    RootKey := HKEY_CURRENT_USER
  else
    RootKey := HKEY_LOCAL_MACHINE;
    
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := RootKey;
    if Reg.OpenKey(RUN_KEY, False) then
    begin
      try
        if Reg.ValueExists(AppName) then
        begin
          Reg.DeleteValue(AppName);
          WriteLn('Auto-start disabled for: ', AppName);
        end
        else
          WriteLn('App not found in auto-start: ', AppName);
      finally
        Reg.CloseKey;
      end;
    end;
  finally
    Reg.Free;
  end;
end;

class function TAutoStart.IsEnabled(const AppName: string; UserLevel: boolean): boolean;
var
  Reg: TRegistry;
  RootKey: HKEY;
begin
  Result := False;
  
  if UserLevel then
    RootKey := HKEY_CURRENT_USER
  else
    RootKey := HKEY_LOCAL_MACHINE;
    
  Reg := TRegistry.Create(KEY_READ);
  try
    Reg.RootKey := RootKey;
    if Reg.OpenKey(RUN_KEY, False) then
    begin
      try
        Result := Reg.ValueExists(AppName);
      finally
        Reg.CloseKey;
      end;
    end;
  finally
    Reg.Free;
  end;
end;

class function TAutoStart.GetStartupPath(const AppName: string; UserLevel: boolean): string;
var
  Reg: TRegistry;
  RootKey: HKEY;
begin
  Result := '';
  
  if UserLevel then
    RootKey := HKEY_CURRENT_USER
  else
    RootKey := HKEY_LOCAL_MACHINE;
    
  Reg := TRegistry.Create(KEY_READ);
  try
    Reg.RootKey := RootKey;
    if Reg.OpenKey(RUN_KEY, False) then
    begin
      try
        if Reg.ValueExists(AppName) then
          Result := Reg.ReadString(AppName);
      finally
        Reg.CloseKey;
      end;
    end;
  finally
    Reg.Free;
  end;
end;

class procedure TAutoStart.ListAllStartupItems;
var
  Reg: TRegistry;
  Names: TStringList;
  Name: string;
  
  procedure ListFromKey(RootKey: HKEY; const Scope: string);
  begin
    Reg.RootKey := RootKey;
    if Reg.OpenKey(RUN_KEY, False) then
    begin
      try
        Names.Clear;
        Reg.GetValueNames(Names);
        
        if Names.Count > 0 then
        begin
          WriteLn('  [', Scope, ']');
          for Name in Names do
            WriteLn('    ', Name, ' = ', Reg.ReadString(Name));
        end;
      finally
        Reg.CloseKey;
      end;
    end;
  end;
  
begin
  Reg := TRegistry.Create(KEY_READ);
  Names := TStringList.Create;
  
  try
    WriteLn('Auto-start items:');
    ListFromKey(HKEY_CURRENT_USER, 'HKCU (Current User)');
    ListFromKey(HKEY_LOCAL_MACHINE, 'HKLM (All Users)');
  finally
    Reg.Free;
    Names.Free;
  end;
end;

var
  TestAppName: string;
  TestAppPath: string;
begin
  WriteLn('=== Auto-Start Management Demo ===');
  WriteLn('');
  
  TestAppName := 'TestAutoStartApp';
  TestAppPath := 'C:\Program Files\TestApp\testapp.exe';
  
  // แสดง auto-start items ปัจจุบัน
  TAutoStart.ListAllStartupItems;
  WriteLn('');
  
  // เพิ่ม auto-start
  WriteLn('Adding to auto-start...');
  TAutoStart.Enable(TestAppName, TestAppPath, True);
  
  WriteLn('');
  WriteLn('Is enabled: ', TAutoStart.IsEnabled(TestAppName, True));
  WriteLn('Startup path: ', TAutoStart.GetStartupPath(TestAppName, True));
  
  // ลบ auto-start
  WriteLn('');
  WriteLn('Removing from auto-start...');
  TAutoStart.Disable(TestAppName, True);
  WriteLn('Is enabled after disable: ', TAutoStart.IsEnabled(TestAppName, True));
end.
{$ELSE}
begin
  WriteLn('Auto-start management is Windows-only');
  WriteLn('On Linux, use systemd service or ~/.config/autostart/');
  WriteLn('On macOS, use LaunchAgents plist in ~/Library/LaunchAgents/');
end.
{$ENDIF}
```

---

## 45.8 File Associations

### การตั้งค่า File Association

```pascal
program FileAssociations;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils, Windows;

type
  TFileAssociation = class
  public
    // ลงทะเบียน file extension
    class procedure Register(
      const Extension: string;      // เช่น '.myext'
      const ProgID: string;          // เช่น 'MyApp.Document'
      const Description: string;    // เช่น 'MyApp Document'
      const IconPath: string;        // Path ไปยัง icon
      const OpenCommand: string      // Command เพื่อเปิดไฟล์
    );
    
    // ยกเลิก file extension
    class procedure Unregister(const Extension: string; const ProgID: string);
    
    // ตรวจสอบว่ามี association อยู่หรือไม่
    class function IsRegistered(const Extension: string): boolean;
    
    // อัพเดท file association cache
    class procedure RefreshShell;
  end;

class procedure TFileAssociation.Register(
  const Extension, ProgID, Description, IconPath, OpenCommand: string);
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CLASSES_ROOT;
    
    // ลงทะเบียน extension
    if Reg.OpenCreateKey(Extension) then
    begin
      Reg.WriteString('', ProgID);
      Reg.CloseKey;
    end;
    
    // สร้าง ProgID
    if Reg.OpenCreateKey(ProgID) then
    begin
      Reg.WriteString('', Description);
      Reg.CloseKey;
    end;
    
    // ตั้งค่า icon
    if IconPath <> '' then
    begin
      if Reg.OpenCreateKey(ProgID + '\DefaultIcon') then
      begin
        Reg.WriteString('', IconPath);
        Reg.CloseKey;
      end;
    end;
    
    // ตั้งค่า open command
    if OpenCommand <> '' then
    begin
      if Reg.OpenCreateKey(ProgID + '\shell\open\command') then
      begin
        Reg.WriteString('', OpenCommand);
        Reg.CloseKey;
      end;
    end;
    
    WriteLn('File association registered for: ', Extension);
    RefreshShell;
    
  finally
    Reg.Free;
  end;
end;

class procedure TFileAssociation.Unregister(const Extension, ProgID: string);
var
  Reg: TRegistry;
begin
  Reg := TRegistry.Create(KEY_ALL_ACCESS);
  try
    Reg.RootKey := HKEY_CLASSES_ROOT;
    Reg.DeleteKey(Extension);
    Reg.DeleteKey(ProgID + '\shell\open\command');
    Reg.DeleteKey(ProgID + '\shell\open');
    Reg.DeleteKey(ProgID + '\shell');
    Reg.DeleteKey(ProgID + '\DefaultIcon');
    Reg.DeleteKey(ProgID);
    WriteLn('File association removed for: ', Extension);
    RefreshShell;
  finally
    Reg.Free;
  end;
end;

class function TFileAssociation.IsRegistered(const Extension: string): boolean;
var
  Reg: TRegistry;
begin
  Result := False;
  Reg := TRegistry.Create(KEY_READ);
  try
    Reg.RootKey := HKEY_CLASSES_ROOT;
    Result := Reg.KeyExists(Extension);
  finally
    Reg.Free;
  end;
end;

class procedure TFileAssociation.RefreshShell;
begin
  // แจ้ง Windows ให้ refresh icon/association cache
  SHChangeNotify(SHCNE_ASSOCCHANGED, SHCNF_IDLIST, nil, nil);
end;

begin
  WriteLn('=== File Association Demo ===');
  WriteLn('');
  
  // ลงทะเบียน .myapp extension
  WriteLn('Registering .myapp file extension...');
  TFileAssociation.Register(
    '.myapp',
    'MyApp.Document',
    'MyApp Document',
    '"C:\Program Files\MyApp\myapp.exe",0',
    '"C:\Program Files\MyApp\myapp.exe" "%1"'
  );
  
  WriteLn('Is .myapp registered: ', TFileAssociation.IsRegistered('.myapp'));
  WriteLn('Is .txt registered: ', TFileAssociation.IsRegistered('.txt'));
  
  // ยกเลิก registration
  WriteLn('');
  WriteLn('Unregistering .myapp...');
  TFileAssociation.Unregister('.myapp', 'MyApp.Document');
  WriteLn('Is .myapp registered now: ', TFileAssociation.IsRegistered('.myapp'));
end.
{$ELSE}
begin
  WriteLn('File associations are Windows-only');
  WriteLn('On Linux, use xdg-mime or freedesktop.org standards');
end.
{$ENDIF}
```

---

## 45.9 Complete Example: App Settings Manager

```pascal
program AppSettingsManager;

{$mode objfpc}{$H+}

{$IFDEF WINDOWS}
uses
  Registry, SysUtils, Classes;

type
  TWindowState = (wsNormal, wsMinimized, wsMaximized);
  
  TUISettings = record
    Theme: string;
    Language: string;
    FontSize: integer;
    ShowToolbar: boolean;
    ShowStatusBar: boolean;
    WindowX, WindowY: integer;
    WindowWidth, WindowHeight: integer;
    WindowState: TWindowState;
  end;
  
  TDatabaseSettings = record
    Server: string;
    Port: integer;
    DatabaseName: string;
    Username: string;
    UseSSL: boolean;
    ConnectionTimeout: integer;
    MaxPoolSize: integer;
  end;
  
  TApplicationSettings = class
  private
    FBaseKey: string;
    FRootKey: HKEY;
    FUI: TUISettings;
    FDatabase: TDatabaseSettings;
    FRecentFiles: TStringList;
    FRecentFolders: TStringList;
    FCustomSettings: TStringList;
    
    procedure LoadUI;
    procedure LoadDatabase;
    procedure LoadRecentFiles;
    procedure LoadCustomSettings;
    
    procedure SaveUI;
    procedure SaveDatabase;
    procedure SaveRecentFiles;
    procedure SaveCustomSettings;
    
    function OpenKey(const SubKey: string; Write: boolean): TRegistry;
  public
    constructor Create(const CompanyName, AppName: string; UserScope: boolean = True);
    destructor Destroy; override;
    
    procedure Load;
    procedure Save;
    procedure Reset;
    procedure Export(const FileName: string);
    procedure Import(const FileName: string);
    
    procedure AddRecentFile(const FilePath: string);
    procedure ClearRecentFiles;
    
    procedure SetCustom(const Key, Value: string);
    function GetCustom(const Key, Default: string): string;
    
    procedure Display;
    
    property UI: TUISettings read FUI write FUI;
    property Database: TDatabaseSettings read FDatabase write FDatabase;
    property RecentFiles: TStringList read FRecentFiles;
  end;

constructor TApplicationSettings.Create(const CompanyName, AppName: string; UserScope: boolean);
begin
  inherited Create;
  
  FBaseKey := 'Software\' + CompanyName + '\' + AppName;
  
  if UserScope then
    FRootKey := HKEY_CURRENT_USER
  else
    FRootKey := HKEY_LOCAL_MACHINE;
  
  FRecentFiles := TStringList.Create;
  FRecentFolders := TStringList.Create;
  FCustomSettings := TStringList.Create;
  
  Reset; // Load defaults
end;

destructor TApplicationSettings.Destroy;
begin
  FRecentFiles.Free;
  FRecentFolders.Free;
  FCustomSettings.Free;
  inherited Destroy;
end;

function TApplicationSettings.OpenKey(const SubKey: string; Write: boolean): TRegistry;
var
  Access: REGSAM;
  FullPath: string;
begin
  if Write then
    Access := KEY_ALL_ACCESS
  else
    Access := KEY_READ;
    
  Result := TRegistry.Create(Access);
  Result.RootKey := FRootKey;
  
  if SubKey <> '' then
    FullPath := FBaseKey + '\' + SubKey
  else
    FullPath := FBaseKey;
  
  if Write then
  begin
    if not Result.OpenCreateKey(FullPath) then
    begin
      Result.Free;
      raise Exception.Create('Cannot open/create registry key: ' + FullPath);
    end;
  end
  else
  begin
    if not Result.OpenKey(FullPath, False) then
    begin
      Result.Free;
      Result := nil;
    end;
  end;
end;

procedure TApplicationSettings.Reset;
begin
  // UI defaults
  FUI.Theme := 'Light';
  FUI.Language := 'th-TH';
  FUI.FontSize := 12;
  FUI.ShowToolbar := True;
  FUI.ShowStatusBar := True;
  FUI.WindowX := 50;
  FUI.WindowY := 50;
  FUI.WindowWidth := 1024;
  FUI.WindowHeight := 768;
  FUI.WindowState := wsNormal;
  
  // Database defaults
  FDatabase.Server := 'localhost';
  FDatabase.Port := 5432;
  FDatabase.DatabaseName := 'myapp';
  FDatabase.Username := 'admin';
  FDatabase.UseSSL := False;
  FDatabase.ConnectionTimeout := 30;
  FDatabase.MaxPoolSize := 10;
  
  FRecentFiles.Clear;
  FCustomSettings.Clear;
end;

procedure TApplicationSettings.LoadUI;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('UI', False);
  if not Assigned(Reg) then Exit;
  
  try
    FUI.Theme := Reg.ReadString('Theme');
    FUI.Language := Reg.ReadString('Language');
    FUI.FontSize := Reg.ReadInteger('FontSize');
    FUI.ShowToolbar := Reg.ReadBool('ShowToolbar');
    FUI.ShowStatusBar := Reg.ReadBool('ShowStatusBar');
    FUI.WindowX := Reg.ReadInteger('WindowX');
    FUI.WindowY := Reg.ReadInteger('WindowY');
    FUI.WindowWidth := Reg.ReadInteger('WindowWidth');
    FUI.WindowHeight := Reg.ReadInteger('WindowHeight');
    FUI.WindowState := TWindowState(Reg.ReadInteger('WindowState'));
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.LoadDatabase;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('Database', False);
  if not Assigned(Reg) then Exit;
  
  try
    FDatabase.Server := Reg.ReadString('Server');
    FDatabase.Port := Reg.ReadInteger('Port');
    FDatabase.DatabaseName := Reg.ReadString('Database');
    FDatabase.Username := Reg.ReadString('Username');
    FDatabase.UseSSL := Reg.ReadBool('UseSSL');
    FDatabase.ConnectionTimeout := Reg.ReadInteger('ConnectionTimeout');
    FDatabase.MaxPoolSize := Reg.ReadInteger('MaxPoolSize');
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.LoadRecentFiles;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('RecentFiles', False);
  if not Assigned(Reg) then Exit;
  
  try
    FRecentFiles.Clear;
    Reg.ReadStringList('Files', FRecentFiles);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.LoadCustomSettings;
var
  Reg: TRegistry;
  Names: TStringList;
  Name: string;
begin
  Reg := OpenKey('Custom', False);
  if not Assigned(Reg) then Exit;
  
  Names := TStringList.Create;
  try
    FCustomSettings.Clear;
    Reg.GetValueNames(Names);
    for Name in Names do
      FCustomSettings.Values[Name] := Reg.ReadString(Name);
  finally
    Reg.CloseKey;
    Reg.Free;
    Names.Free;
  end;
end;

procedure TApplicationSettings.Load;
begin
  Reset; // load defaults first
  
  try
    LoadUI;
    LoadDatabase;
    LoadRecentFiles;
    LoadCustomSettings;
    WriteLn('Settings loaded from registry');
  except
    on E: Exception do
      WriteLn('Warning: Could not load settings: ', E.Message);
  end;
end;

procedure TApplicationSettings.SaveUI;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('UI', True);
  try
    Reg.WriteString('Theme', FUI.Theme);
    Reg.WriteString('Language', FUI.Language);
    Reg.WriteInteger('FontSize', FUI.FontSize);
    Reg.WriteBool('ShowToolbar', FUI.ShowToolbar);
    Reg.WriteBool('ShowStatusBar', FUI.ShowStatusBar);
    Reg.WriteInteger('WindowX', FUI.WindowX);
    Reg.WriteInteger('WindowY', FUI.WindowY);
    Reg.WriteInteger('WindowWidth', FUI.WindowWidth);
    Reg.WriteInteger('WindowHeight', FUI.WindowHeight);
    Reg.WriteInteger('WindowState', Ord(FUI.WindowState));
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.SaveDatabase;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('Database', True);
  try
    Reg.WriteString('Server', FDatabase.Server);
    Reg.WriteInteger('Port', FDatabase.Port);
    Reg.WriteString('Database', FDatabase.DatabaseName);
    Reg.WriteString('Username', FDatabase.Username);
    Reg.WriteBool('UseSSL', FDatabase.UseSSL);
    Reg.WriteInteger('ConnectionTimeout', FDatabase.ConnectionTimeout);
    Reg.WriteInteger('MaxPoolSize', FDatabase.MaxPoolSize);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.SaveRecentFiles;
var
  Reg: TRegistry;
begin
  Reg := OpenKey('RecentFiles', True);
  try
    Reg.WriteStringList('Files', FRecentFiles);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.SaveCustomSettings;
var
  Reg: TRegistry;
  i: integer;
begin
  Reg := OpenKey('Custom', True);
  try
    for i := 0 to FCustomSettings.Count - 1 do
      Reg.WriteString(FCustomSettings.Names[i], FCustomSettings.ValueFromIndex[i]);
  finally
    Reg.CloseKey;
    Reg.Free;
  end;
end;

procedure TApplicationSettings.Save;
begin
  SaveUI;
  SaveDatabase;
  SaveRecentFiles;
  SaveCustomSettings;
  WriteLn('Settings saved to registry');
end;

procedure TApplicationSettings.Export(const FileName: string);
var
  Lines: TStringList;
  i: integer;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('[UI]');
    Lines.Add('Theme=' + FUI.Theme);
    Lines.Add('Language=' + FUI.Language);
    Lines.Add('FontSize=' + IntToStr(FUI.FontSize));
    Lines.Add('ShowToolbar=' + BoolToStr(FUI.ShowToolbar));
    Lines.Add('ShowStatusBar=' + BoolToStr(FUI.ShowStatusBar));
    Lines.Add('');
    Lines.Add('[Database]');
    Lines.Add('Server=' + FDatabase.Server);
    Lines.Add('Port=' + IntToStr(FDatabase.Port));
    Lines.Add('Database=' + FDatabase.DatabaseName);
    Lines.Add('');
    Lines.Add('[RecentFiles]');
    for i := 0 to FRecentFiles.Count - 1 do
      Lines.Add('File' + IntToStr(i + 1) + '=' + FRecentFiles[i]);
    Lines.Add('');
    Lines.Add('[Custom]');
    for i := 0 to FCustomSettings.Count - 1 do
      Lines.Add(FCustomSettings[i]);
    
    Lines.SaveToFile(FileName);
    WriteLn('Settings exported to: ', FileName);
  finally
    Lines.Free;
  end;
end;

procedure TApplicationSettings.Import(const FileName: string);
begin
  WriteLn('Import from: ', FileName, ' (not implemented in demo)');
end;

procedure TApplicationSettings.AddRecentFile(const FilePath: string);
var
  Idx: integer;
begin
  Idx := FRecentFiles.IndexOf(FilePath);
  if Idx >= 0 then
    FRecentFiles.Delete(Idx);
  FRecentFiles.Insert(0, FilePath);
  
  // เก็บแค่ 10 ไฟล์ล่าสุด
  while FRecentFiles.Count > 10 do
    FRecentFiles.Delete(FRecentFiles.Count - 1);
end;

procedure TApplicationSettings.ClearRecentFiles;
begin
  FRecentFiles.Clear;
end;

procedure TApplicationSettings.SetCustom(const Key, Value: string);
begin
  FCustomSettings.Values[Key] := Value;
end;

function TApplicationSettings.GetCustom(const Key, Default: string): string;
var
  Idx: integer;
begin
  Idx := FCustomSettings.IndexOfName(Key);
  if Idx >= 0 then
    Result := FCustomSettings.ValueFromIndex[Idx]
  else
    Result := Default;
end;

procedure TApplicationSettings.Display;
begin
  WriteLn('=== Application Settings ===');
  WriteLn('');
  WriteLn('[ UI ]');
  WriteLn('  Theme: ', FUI.Theme);
  WriteLn('  Language: ', FUI.Language);
  WriteLn('  FontSize: ', FUI.FontSize);
  WriteLn('  Toolbar: ', BoolToStr(FUI.ShowToolbar, 'Shown', 'Hidden'));
  WriteLn('  Window: ', FUI.WindowWidth, 'x', FUI.WindowHeight, 
    ' at (', FUI.WindowX, ',', FUI.WindowY, ')');
  WriteLn('');
  WriteLn('[ Database ]');
  WriteLn('  Server: ', FDatabase.Server, ':', FDatabase.Port);
  WriteLn('  Database: ', FDatabase.DatabaseName);
  WriteLn('  Username: ', FDatabase.Username);
  WriteLn('  SSL: ', BoolToStr(FDatabase.UseSSL, 'Enabled', 'Disabled'));
  WriteLn('');
  if FRecentFiles.Count > 0 then
  begin
    WriteLn('[ Recent Files ]');
    var i: integer;
    for i := 0 to FRecentFiles.Count - 1 do
      WriteLn('  ', i + 1, '. ', FRecentFiles[i]);
    WriteLn('');
  end;
  if FCustomSettings.Count > 0 then
  begin
    WriteLn('[ Custom ]');
    var i: integer;
    for i := 0 to FCustomSettings.Count - 1 do
      WriteLn('  ', FCustomSettings[i]);
  end;
end;

var
  Settings: TApplicationSettings;
  ExportFile: string;
begin
  WriteLn('=== App Settings Manager Demo ===');
  WriteLn('');
  
  ExportFile := GetTempDir + 'app_settings.ini';
  
  Settings := TApplicationSettings.Create('MyCompany', 'MyApp');
  try
    // กำหนดค่า
    Settings.UI.Theme := 'Dark';
    Settings.UI.Language := 'th-TH';
    Settings.UI.FontSize := 14;
    Settings.UI.WindowWidth := 1400;
    Settings.UI.WindowHeight := 900;
    
    Settings.Database.Server := 'production.db.mycompany.com';
    Settings.Database.Port := 5432;
    Settings.Database.DatabaseName := 'myapp_production';
    Settings.Database.Username := 'app_readonly';
    Settings.Database.UseSSL := True;
    
    Settings.AddRecentFile('C:\Projects\project1.myapp');
    Settings.AddRecentFile('D:\Documents\report2024.myapp');
    Settings.AddRecentFile('C:\Projects\project2.myapp');
    
    Settings.SetCustom('LastBackupDate', '2024-01-15');
    Settings.SetCustom('LicenseKey', 'XXXX-YYYY-ZZZZ');
    Settings.SetCustom('UpdateChannel', 'stable');
    
    // แสดงและบันทึก
    Settings.Display;
    
    Settings.Save;
    Settings.Export(ExportFile);
    
    WriteLn('');
    WriteLn('Settings saved!');
    
    // ทดสอบโหลดกลับ
    WriteLn('');
    WriteLn('Reloading settings from registry...');
    Settings.Load;
    Settings.Display;
    
  finally
    Settings.Free;
    
    // Cleanup registry
    var Reg := TRegistry.Create(KEY_ALL_ACCESS);
    try
      Reg.RootKey := HKEY_CURRENT_USER;
      Reg.DeleteKey('Software\MyCompany\MyApp\UI');
      Reg.DeleteKey('Software\MyCompany\MyApp\Database');
      Reg.DeleteKey('Software\MyCompany\MyApp\RecentFiles');
      Reg.DeleteKey('Software\MyCompany\MyApp\Custom');
      Reg.DeleteKey('Software\MyCompany\MyApp');
      Reg.DeleteKey('Software\MyCompany');
    finally
      Reg.Free;
    end;
    
    if FileExists(ExportFile) then
      DeleteFile(ExportFile);
  end;
end.
{$ELSE}
begin
  WriteLn('This demo requires Windows');
  WriteLn('');
  WriteLn('On non-Windows platforms, consider using:');
  WriteLn('  - TIniFile for simple settings');
  WriteLn('  - JSON/XML files for complex settings');
  WriteLn('  - SQLite for database-backed settings');
end.
{$ENDIF}
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: Registry Backup/Restore
สร้างเครื่องมือ backup และ restore registry keys:
- Export registry tree เป็น .reg format
- Import จาก .reg file
- รองรับ HKCU และ HKLM

### ข้อ 2: Multi-User Settings
ระบบ settings ที่รองรับหลาย users:
- Settings แยกต่างหากสำหรับแต่ละ user
- Inherit default settings จาก machine-level
- Override ใน user-level

### ข้อ 3: Registry Search Tool
เครื่องมือค้นหาใน Registry:
- ค้นหา key, value name, หรือ value data
- Regular expression support
- Export ผลลัพธ์เป็น text

### ข้อ 4: Settings Migration
Migration tool สำหรับเปลี่ยน settings format:
- อ่าน settings เก่าจาก INI file
- เขียน settings ใหม่ลง Registry
- ยืนยันความถูกต้อง

### ข้อ 5: Registry Audit
Audit tool สำหรับ monitor การเปลี่ยนแปลง:
- บันทึก snapshot ของ registry section
- Compare กับ snapshot เก่า
- Report changes (added/modified/deleted)
